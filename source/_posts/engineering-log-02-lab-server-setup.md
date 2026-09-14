---
title: 工程日志 02：实验室服务器的远程开发环境配置
date: 2026-09-14 11:29:35
categories: Engineering
tags:
  - 工程日志
  - Linux
  - SSH
  - 代理
  - Conda
---

记录课题组 GPU 服务器的远程开发环境配置：SSH 免密与别名、VS Code 接入、共享 GPU 与 conda 的现状、GitHub 不可达时的代理方案、镜像与缓存、tmux 与目录约定。访问链路前面还有一层 VPN，与本文无关，略过。下文中 `<SERVER>`、`<USER>`、`<PROXY_PORT>` 是占位符。

<!-- more -->

## 环境

| 项目 | 值 |
|---|---|
| 本地 | macOS 15.7，Apple Silicon |
| 服务器 | Ubuntu 22.04.5，内核 6.8 |
| GPU | 8 × RTX A6000（48 GB / 卡） |
| 驱动 / CUDA | 570.172.08 / 12.8 |
| 网络 | 服务器有外网，GitHub 的 TLS 连接会被中断 |
| 权限 | 普通用户，无 sudo |

两个前提决定了后面的取舍：没有 sudo，所以任何需要写系统目录的方案都排除；机器共享，配置不能影响别人正在跑的任务。

## SSH 免密与别名

```bash
ssh-keygen -t ed25519          # 已有密钥则跳过
ssh-copy-id -p 22 <USER>@<SERVER>
```

把本地公钥追加到服务器的 `~/.ssh/authorized_keys`，之后握手由服务器出题、本地私钥作答，私钥不离开本地。

本地 `~/.ssh/config`：

```ini
Host lab
    HostName <SERVER>
    User <USER>
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 6
```

`ssh lab` 即可登录。后两行每 30 秒发一次心跳，连续 6 次无响应才断开；远程开发时长时间不敲键盘，中间的网络设备容易回收空闲连接，加上之后基本不再出现会话卡死。

## VS Code 接入

装 Remote-SSH，连接 `lab`，首次连接会自动在服务器上部署 VS Code Server。

需要注意插件装在哪一侧。Remote-SSH 把 VS Code 拆成两半，界面在本地、语言服务和调试器在服务器。主题、键位这类装本地；Python、Jupyter、Pylance 这类要读代码文件、调用解释器，必须装在服务器端，安装按钮会显示 `Install in SSH: lab`。

另外 Python 插件不是 Python。解释器是 conda 环境里的 `python`，负责执行；插件负责补全、跳转、断点，以及在状态栏切换解释器。打开 `.py` 文件后确认右下角显示的解释器路径是自己的环境。

TensorBoard 和 Jupyter 用 VS Code 的端口面板转发 6006 / 8888，本地浏览器直接打开。

## GPU 与 conda 现状

```bash
nvidia-smi
```

关注 `Memory-Usage`、`GPU-Util` 和下半部分的 `Processes`。登录时 8 张卡里有 5 张在跑别人的任务。共享机器上每次跑之前先看哪张卡空着，用环境变量指定，不要动别人的进程：

```bash
CUDA_VISIBLE_DEVICES=1 python train.py
```

conda 的情况：

```bash
conda env list
# base         /opt/anaconda3
# pytorch_env  /opt/anaconda3/envs/pytorch_env
```

管理员已装了全局 conda，不需要再往家目录装 miniconda。`/opt/anaconda3` 是 root 所有的，写不进去，所以不能往 `base` 和公共的 `pytorch_env` 里装包。conda 检测到 `/opt` 不可写时会把新环境放到 `~/.conda/envs/`，正好是需要的行为。

```bash
conda init bash
conda config --set auto_activate_base false
conda create -n py310-myproj python=3.10 -y
conda env list        # 确认新环境落在 ~/.conda/envs 而非 /opt
```

## GitHub 不可达

```
$ git clone https://github.com/<owner>/<repo>.git
fatal: unable to access 'https://github.com/<owner>/<repo>.git/':
GnuTLS recv error (-110): The TLS connection was non-properly terminated.
```

TLS 握手中途连接被重置，不是正常关闭。解析和建连都成功，问题出在加密协商阶段。具体是哪一层设备做的，从客户端无法证实，但不影响处理方式。

先确认是 GitHub 单独不通还是整机没有外网：

```bash
curl -sI --max-time 10 https://pypi.tuna.tsinghua.edu.cn | head -1   # 通则有外网
curl -sI --max-time 10 https://github.com | head -1
```

四种方案：问管理员要课题组的代理；用镜像站；本地克隆后 rsync 上去；把本地已有的代理通过 SSH 隧道借给服务器。前三种分别受限于等待、镜像站寿命和每次手动搬运，选了第四种，不需要在服务器上装任何东西，也不需要额外权限。

原理是 SSH 反向端口转发：在服务器上监听一个端口，流量沿已建立的 SSH 隧道回到本地，从本地代理出去。在 `Host lab` 段里加一行：

```ini
    RemoteForward <PROXY_PORT> 127.0.0.1:<PROXY_PORT>
```

`<PROXY_PORT>` 是本地代理客户端的混合端口，在客户端设置里能查到，同时支持 HTTP 和 SOCKS。端口绑在服务器的 `127.0.0.1` 上，内网其他人访问不到。

配置只对新连接生效，断开重连后让 git 走它。只对 GitHub 生效，避免访问国内站点时绕路：

```bash
git config --global http.https://github.com.proxy http://127.0.0.1:<PROXY_PORT>
curl -sI --max-time 10 -x http://127.0.0.1:<PROXY_PORT> https://github.com | head -1
git clone https://github.com/<owner>/<repo>.git
```

VS Code 的 Remote-SSH 读同一份 `~/.ssh/config`，内置终端里同样可用。

三个前提：本地开机、代理运行、VPN 连着。同时开多个 SSH 会话时，第二个会提示 `remote port forwarding failed for listen port <PROXY_PORT>`，端口已被占用，忽略即可。另外 tmux 里的任务在 SSH 断开后继续运行，但此时代理已失效，下载类操作要在连接期间做完。

## 镜像与缓存

装包用代理是绕远路，直接走国内镜像：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

`~/.bashrc` 里加模型镜像和缓存目录。默认缓存都在家目录，模型权重几十 GB 很容易把配额占满：

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HOME=~/.cache/huggingface
export TORCH_HOME=~/.cache/torch
export PIP_CACHE_DIR=~/.cache/pip
```

`df -h` 看清哪块盘容量大，之后把 `~/.cache` 软链过去。

## tmux

SSH 会话结束时其中的进程会被终止，长任务放 tmux：

```bash
tmux new -s exp1          # Ctrl+B D 脱离
tmux attach -t exp1
tmux ls
```

本地已有的配置直接复制过去：

```bash
scp ~/.tmux.conf lab:~/.tmux.conf
tmux kill-server 2>/dev/null    # 旧会话不会重读配置
```

复制前确认两点：配置里有没有依赖 tpm 或 macOS 专用命令（`pbcopy`、`reattach-to-user-namespace`），这些在 Linux 上不生效；两端 `tmux -V` 版本差距大时有少量配置项改过名。前缀键若改成 `Ctrl+A`，bash 的行首跳转要连按两次。

## 目录约定

```
~/
├── code/                 # git 仓库，只放代码
├── data/      -> 大盘     # 数据集，只读，多项目共用
├── outputs/   -> 大盘     # checkpoint、日志、生成结果
│   └── <项目>/20260914_lr1e-4_bs32/
└── .cache/    -> 大盘
```

代码、数据、产出分开，因为三者生命周期不同：代码进 git 可随时重新克隆，数据只读且多项目共用，产出体积最大且可再生。`.gitignore` 里要有 `outputs/`、`data/`、`*.pt`、`*.ckpt`、`wandb/`，checkpoint 一旦进了 `.git` 仓库就没法收拾。

家目录一般在系统盘上，软链目标先用 `df -h` 确认，有共享数据盘的话问一下约定再动手。每次实验的产出目录里留好配置、日志、checkpoint 和当次代码的 commit hash。

## 可复用片段

本地 `~/.ssh/config`：

```ini
Host lab
    HostName <SERVER>
    User <USER>
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 6
    RemoteForward <PROXY_PORT> 127.0.0.1:<PROXY_PORT>
```

服务器端初始化：

```bash
#!/usr/bin/env bash
set -euo pipefail

conda init bash
conda config --set auto_activate_base false

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
cat >> ~/.bashrc <<'EOF'
export HF_ENDPOINT=https://hf-mirror.com
export HF_HOME=~/.cache/huggingface
export TORCH_HOME=~/.cache/torch
export PIP_CACHE_DIR=~/.cache/pip
EOF

git config --global http.https://github.com.proxy http://127.0.0.1:<PROXY_PORT>
mkdir -p ~/code ~/outputs ~/.cache
```

## 未尽事项

数据盘还没确认，`data/` 和 `outputs/` 的软链目标待定。代理依赖本地机器开着，课题组若有自己的代理应当换过去。实验记录仍是手工的，wandb 或 TensorBoard 要在第一个正式实验前接上。`environment.yml` 等环境稳定后再导出。
