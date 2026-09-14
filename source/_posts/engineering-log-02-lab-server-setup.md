---
title: 工程日志 02：实验室服务器环境与代理配置全记录
date: 2026-09-14 11:29:35
categories: Engineering
tags:
  - 工程日志
  - Linux
  - SSH
  - VPN
  - 代理
  - Conda
---

从拿到一行「VPN 连上，server 是 203.0.113.10:22」开始，到能在 VS Code 里编辑服务器上的代码、在 tmux 里跑训练、从 GitHub 克隆仓库为止。中间卡了四次：VPN 客户端装不上、VPN 账号登不进、SSH 第一次被拒、GitHub 克隆被掐断。这篇把每一步的做法、原因和验证方式都记下来，也把四个报错的原文和根因写清楚。读者假设是第一次接触远程服务器的人。

<!-- more -->

## 背景与目标

课题组有一台 8 卡 GPU 服务器，跑实验要用它。导师给的全部信息就两句话：VPN 的使用说明在压缩包里，服务器地址是 `203.0.113.10:22`，VPN 的账号密码和服务器账号密码相同。

这两句话里藏着一个新手最容易卡住的地方：**这是两层，不是一层**。

- **第一层是 VPN**：用 Cisco Secure Client 连到课题组的网关，作用是把你的笔记本"拉进"内网。没有这一步，服务器的内网地址在公网上根本不存在。
- **第二层是 SSH**：进了内网之后，用终端登录 `203.0.113.10` 的 22 端口。22 是 SSH 的标准端口，不是什么"VPN 端口"。

所以 VPN 客户端界面里**没有地方填服务器地址**，这是正常的，它只管第一层。想明白这件事之前，我一直在 VPN 客户端里找"添加服务器"的按钮，还差点跑去 macOS 的「系统设置 → 网络 → VPN」里手工配一个 Cisco IPSec 连接——那是 macOS 自带的另一种老式 VPN 协议，跟这个客户端毫无关系，把服务器 IP 填进去也连不上。

目标状态：本地 VS Code 打开的就是服务器上的目录，改文件即改服务器文件，终端、调试器、Jupyter 全在服务器上跑；GPU 能用；GitHub、pip、HuggingFace 都能正常拉东西。

## 环境概况

| 项目 | 值 |
|---|---|
| 本地 | macOS 15.7，Apple Silicon |
| 服务器系统 | Ubuntu 22.04.5 LTS，内核 6.8 |
| GPU | 8 × NVIDIA RTX A6000（48 GB 显存 / 卡） |
| 驱动 / CUDA | 570.172.08 / CUDA 12.8 |
| VPN 客户端 | Cisco Secure Client 5.0.01242 |
| 网络限制 | 服务器有外网，但访问 GitHub 的 TLS 连接会被中断 |
| 账号权限 | 普通用户，无 sudo |

有两个前提决定了后面所有决策：**没有 sudo**，所以任何需要写系统目录的方案都不可行；**机器是共享的**，配置时不能影响别人正在跑的任务。

## 一、装 VPN 客户端：一次失败的重装

客户端第一次装是成功的。后来我用一个第三方卸载工具（App Cleaner & Uninstaller 这类）把它卸了，卸载过程中弹出一个提示说某个文件夹没有权限删除，我没管。再装回去时，安装器走到最后一步报了：

```
安装失败。
安装器遇到了一个错误，导致安装失败。请联系软件生产企业以获得帮助。
```

这个提示等于什么都没说。真正有用的信息在系统的安装日志里：

```bash
sudo grep -i cisco /var/log/install.log | tail -50
```

关键的三行是：

```
./postinstall: Attempting to start the VPN agent ...
./postinstall: Bootstrap failed: 5: Input/output error
PackageKit: Install Failed: Error Domain=PKInstallErrorDomain Code=112
```

`Code=112` 的意思是安装包的脚本执行失败。文件其实已经全部复制到位了，失败在最后一步——启动 VPN 后台服务。

根因是那个第三方卸载工具。它卸载时不只是删文件，还把两个后台服务在 launchd（macOS 的服务管理器）里**永久标记为禁用**。这个禁用状态写在系统数据库里，删文件和重启都清不掉。查一下就能看到：

```bash
launchctl print system | grep -i cisco
# "com.cisco.secureclient.vpnagentd" => disabled
# "com.cisco.secureclient.ciscod64"  => disabled
```

服务被禁用时，安装脚本再去启动它就会得到 `Input/output error`，整个安装被判失败。解法是把禁用状态解除，再重装：

```bash
sudo launchctl enable system/com.cisco.secureclient.vpnagentd
sudo launchctl enable system/com.cisco.secureclient.ciscod64
launchctl print-disabled system | grep -i cisco    # 应无任何输出
```

顺带说那个"删不掉的文件夹"。它在 `/Library/SystemExtensions/` 下，是客户端的网络过滤**系统扩展**。这个目录受 SIP（系统完整性保护）保护，访达点删除、终端 `sudo rm`，全都会被拒绝——这是 macOS 的设计，不是权限没给够。孤儿扩展不用管，重装时会被新安装接管。

**经验**：带系统扩展的软件（VPN、杀毒、网络工具）不要用第三方清理工具卸载。系统扩展的注销必须由软件自己发起，第三方工具先把主程序删了，扩展就成了孤儿，服务还被顺手禁用。这类软件基本都自带卸载器，用它：

```bash
# 官方卸载方式，二选一
open "/Applications/Cisco/Uninstall Cisco Secure Client.app"
sudo /opt/cisco/secureclient/bin/vpn_uninstall.sh
```

重装时在「安装类型」页点「自定」，只勾 VPN，其余模块（Umbrella、Secure Endpoint、Posture、ISE Posture、DART）全部取消——勾得越多失败点越多，日常连内网也用不到。安装中途会弹「系统扩展已被阻止」，**不要关掉安装器**，去「系统设置 → 隐私与安全性」点允许，再回来等它完成。装完验证：

```bash
systemextensionsctl list | grep -i cisco     # 状态应是 [activated enabled]
```

## 二、VPN 登录：账号问题就是账号问题

客户端里填网关地址（形如 `vpn.example.edu:<PORT>`，在课题组给的说明里）然后点连接，会弹出一个要求填「组 / 用户名 / 密码」的框。组是一个下拉列表，列着网关上配置的所有分组。

这里我卡了很久，每个组都试了一遍，全部返回：

```
登陆失败: 用户名或密码错误
```

后来想明白，这个现象本身已经说明了很多事：**能弹出组列表，说明网关已经连通了**。网络、客户端、地址全都没问题，它只是在最后一步核对账号时说"不认识你"。也就是说，这不是配置问题，不是客户端问题，更不是 macOS 系统设置里那个 VPN 没打开——它纯粹是账号问题，而账号问题只能找管理员解决。

排查顺序：

1. **组别选错**。列表里往往按导师或课题组分组，要选自己所属的那个。像 `ops` 这种通常是运维组，不给学生用。
2. **账号还没开通**。"VPN 账号密码和服务器账号相同"这句话的潜台词是，管理员需要把服务器账号同步到 VPN 上，可能还没同步。
3. **输入本身有问题**。从聊天软件复制密码容易带上末尾空格，用户名大小写也要完全一致。

我这边最后是换了一个账号才登进去的。写给管理员的问法可以直接抄：

> Cisco 客户端已装好并能连上网关，会弹出组别选择框。但选任何一个组、输入服务器账号和密码，都提示「用户名或密码错误」。请问应该选哪个组？我的 VPN 账号是不是还没开通？

**要记住的**：VPN 必须全程保持连接。客户端窗口可以关掉，它会缩到菜单栏继续运行，看图标状态即可。合盖睡眠后 VPN 通常会断，醒来要重连。

## 三、SSH 首次登录：那句吓人的提示是正常的

VPN 连上后，本地终端：

```bash
ssh <user>@203.0.113.10 -p 22
```

第一次的结果是：

```
Connection closed by 203.0.113.10 port 22
```

服务器在握手阶段直接把连接关掉了，还没走到问密码那一步。隔几秒重试就好了，最可能的原因是 VPN 刚连上时路由还没完全建好。（这一条我没有拿到服务器端日志，不能完全确证，也有可能是服务端的防暴力破解机制短暂拦了一下。）

重试之后出现了这段，很多人看到会停手：

```
The authenticity of host '203.0.113.10' can't be established.
ED25519 key fingerprint is SHA256:<FINGERPRINT>.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

**这不是报错，这是 SSH 第一次连接任何新机器时的固定提示。** 每台 SSH 服务器都有一对身份密钥，那串 SHA256 是它公钥的指纹。你的电脑从没见过这台机器，没法替你验证"对面确实是它"，所以停下来问你。

这个机制防的是中间人攻击：有人在网络上冒充服务器骗你的密码。严格做法是找管理员核对指纹。内网加 VPN 的场景下风险很低，输 `yes` 即可：

```
Warning: Permanently added '203.0.113.10' (ED25519) to the list of known hosts.
```

指纹被写进了本地的 `~/.ssh/known_hosts`，以后不会再问。**反过来，如果哪天再连时弹出大段 `REMOTE HOST IDENTIFICATION HAS CHANGED` 的警告，那才要警惕**——要么服务器重装了系统，要么真有人在冒充，这时先问管理员再连。

登录后的欢迎横幅里有一堆 `105 updates can be applied`、`*** System restart required ***` 之类的提示，那是给管理员看的运维信息，普通用户既没权限也不应该去执行——共享服务器上正跑着别人的训练任务，重启它是事故。

## 四、免密登录与 SSH 配置

每次输密码很烦，而且 VS Code 会频繁建立连接。配一次密钥认证：

```bash
ssh-keygen -t ed25519            # 已有密钥就跳过，一路回车
ssh-copy-id -p 22 <user>@203.0.113.10   # 输最后一次密码
```

原理是把本地的**公钥**追加到服务器的 `~/.ssh/authorized_keys`。之后握手时服务器用公钥出一道只有对应私钥能解的题，私钥全程不离开本地。

然后在本地写 `~/.ssh/config`，给这台机器起个别名：

```ini
Host lab
    HostName 203.0.113.10
    User <user>
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 6
```

之后 `ssh lab` 就能直接登录。后两行的作用是每 30 秒发一个心跳包，连续 6 次无响应才断开——远程开发时长时间不敲键盘，中间的网络设备容易悄悄掐掉空闲连接，这两行能显著减少"回来发现终端卡死"的情况。

验证：`ssh lab` 应该不问密码直接进入。

## 五、VS Code 接入：插件装在哪一边

装 **Remote - SSH** 插件，左下角绿色按钮 → Connect to Host → 选 `lab`。第一次连接时 VS Code 会自动在服务器上安装一个 VS Code Server，之后 File → Open Folder 打开服务器上的目录。

这里有个概念要分清。Remote-SSH 模式下 VS Code 被拆成两半：**界面这一半在本地跑，干活那一半在服务器上跑**。插件也因此分两类：

- 主题、快捷键这类只影响界面的，装在本地。
- Python、Jupyter、Pylance 这类需要读代码文件、调用解释器、跑调试器的，**必须装在服务器端**。装的时候按钮会显示「Install in SSH: lab」，点它。

顺带澄清另一个容易混的点：**VS Code 的 Python 插件不是 Python**。解释器是服务器上 conda 环境里那个 `python`，真正执行代码；插件只负责语法高亮、补全、跳转、断点调试，以及在右下角让你选用哪个解释器。两者缺一不可——插件没有解释器跑不了代码，解释器没有插件就只能在终端敲命令。

打开一个 `.py` 文件后看右下角状态栏显示的解释器路径，不对就点它切换。

想看 TensorBoard 或 Jupyter，用 VS Code 的「端口」面板转发 6006 / 8888，本地浏览器直接打开即可，不需要额外配置。

## 六、认识这台机器：共享 GPU 的礼仪

登录后第一件事是看显卡：

```bash
nvidia-smi
```

输出里要看三列。`Memory-Usage` 是显存占用，`GPU-Util` 是利用率，下半部分的 `Processes` 列出每张卡上正在跑的进程。我登录时的情况是：0、3、4、5、6 号卡都在 90% 以上跑着别人的 Python 进程，1、2、7 号空闲。

**这是一台多人共享的机器，规矩比技术重要**：每次跑之前先 `nvidia-smi` 看哪张卡空着，只用空闲的卡，永远不要 kill 不属于自己的进程。指定显卡靠环境变量：

```bash
CUDA_VISIBLE_DEVICES=1 python train.py        # 只用 1 号卡
CUDA_VISIBLE_DEVICES=1,2 python train.py      # 用 1、2 两张
```

再看 Python 环境：

```bash
conda env list
# base                     /opt/anaconda3
# pytorch_env              /opt/anaconda3/envs/pytorch_env
which conda
# /opt/anaconda3/bin/conda
```

管理员已经装了全局 conda，**不需要**再往自己家目录装一份 miniconda。但 `/opt/anaconda3` 是 root 所有的，普通用户写不进去，这意味着：不能往 `base` 或公共的 `pytorch_env` 里 `pip install` 任何东西，也不该去改它们。

好消息是 conda 检测到 `/opt` 不可写时，会自动把新建的环境放到 `~/.conda/envs/`，包缓存放到 `~/.conda/pkgs/`，正好是想要的行为。先做一次 shell 初始化，否则 `conda activate` 会报错：

```bash
conda init bash
source ~/.bashrc
conda config --set auto_activate_base false     # 可选，不自动进 base

conda create -n py310-myproj python=3.10 -y
conda env list        # 新环境路径应是 ~/.conda/envs/py310-myproj，确认没写进 /opt
conda activate py310-myproj
```

## 七、GitHub 被掐断：把本地代理转发给服务器

建好目录准备克隆仓库，撞上这个：

```
$ git clone https://github.com/<owner>/<repo>.git
Cloning into '<repo>'...
fatal: unable to access 'https://github.com/<owner>/<repo>.git/':
GnuTLS recv error (-110): The TLS connection was non-properly terminated.
```

这句话的意思是：TLS 握手进行到一半，连接被**强制中断**了（收到 RST），不是正常关闭。不是 git 的 bug，不是仓库不存在，也不是 DNS 解析失败——解析和建连都成功了，是在加密协商阶段被打断的。服务器本身有外网，只是一碰 GitHub 就被掐。

（这类中断的具体是哪一层设备做的，从客户端这边无法证实，只能说特征吻合。对解决方案没有影响。）

先花一分钟确认是"GitHub 不通"还是"整机没网"，这决定了后面走哪条路：

```bash
curl -sI --max-time 10 https://pypi.tuna.tsinghua.edu.cn | head -1   # 国内站点，通 → 有外网
curl -sI --max-time 10 https://github.com | head -1                  # 大概率超时或被重置
```

**方案选择。** 想到过四条路：

1. 问管理员要课题组的代理——最省事，但要等回复。
2. 用 GitHub 镜像站——速度看运气，镜像站生命周期短，今天能用明天可能就没了。
3. 在本地克隆完再 `rsync` 上去——一定有效，但每次更新仓库都要手动搬一次。
4. **把本地已有的代理通过 SSH 隧道借给服务器**——不需要在服务器上装任何东西，不需要任何额外权限，一次配置长期有效。

选了第四条。原理是 SSH 的**反向端口转发**：在服务器上开一个监听端口，任何连它的流量都通过已经建好的 SSH 隧道回到本地机器，从本地的代理客户端出去。方向和平时的端口转发相反——平时是把远程的端口拿到本地看，这次是把本地的能力送到远程去。

在本地 `~/.ssh/config` 的 `Host lab` 段里加一行：

```ini
Host lab
    HostName 203.0.113.10
    User <user>
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 6
    RemoteForward <PROXY_PORT> 127.0.0.1:<PROXY_PORT>
```

`<PROXY_PORT>` 换成本地代理客户端的「混合端口 / Mixed Port」，在客户端设置界面里能查到。这个端口同时支持 HTTP 和 SOCKS 协议，所以一个就够。注意它绑定在服务器的 `127.0.0.1` 上，只有服务器本机能访问，不会暴露给内网其他人。

配置只在**新建立的连接**上生效，所以要断开重连 `ssh lab`。然后让 git 走这个代理——只对 GitHub 生效，这样访问国内镜像时不会绕远路：

```bash
git config --global http.https://github.com.proxy http://127.0.0.1:<PROXY_PORT>
```

验证与克隆：

```bash
curl -sI --max-time 10 -x http://127.0.0.1:<PROXY_PORT> https://github.com | head -1
# 预期：HTTP/1.1 200 Connection established  或  HTTP/2 200
git clone https://github.com/<owner>/<repo>.git
```

VS Code 的 Remote-SSH 读的是同一份 `~/.ssh/config`，所以它连上去时也会自动建好这个转发，在 VS Code 内置终端里一样能用。

**三个必须知道的前提**：本地机器要开机、代理客户端要运行、VPN 要连着，三者缺一不可。另外如果同时开了多个 SSH 会话，第二个会提示 `Warning: remote port forwarding failed for listen port <PROXY_PORT>`，这是因为端口已被第一个会话占用，不影响使用，忽略即可。最后一条最容易忘：**tmux 里的长任务在 SSH 断开后仍在跑，但那时代理已经没了**，所以下载类操作要在 SSH 连着的时候做完。

## 八、包管理走国内镜像

代理能解决 GitHub，但拿它装 pip 包是舍近求远——所有流量要绕一圈回到本地机器再出去，慢且脆弱。pip 和模型权重直接用国内镜像更快：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

HuggingFace 下模型权重，在 `~/.bashrc` 里加一行，速度和稳定性都远好于走代理：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

顺便把缓存目录也在 `~/.bashrc` 里定向一下。HuggingFace 和 torch 下载的模型动辄几十 GB，默认全堆在家目录，很容易把配额撑爆：

```bash
export HF_HOME=~/.cache/huggingface
export TORCH_HOME=~/.cache/torch
export PIP_CACHE_DIR=~/.cache/pip
```

如果服务器上挂了大容量数据盘（`df -h` 看哪块盘大），更好的做法是把 `~/.cache` 整个软链过去。

## 九、tmux：让训练熬过断线

VPN 一断、笔记本一合盖，SSH 会话就没了，在它里面跑的训练进程会跟着被杀掉。tmux 的作用是把进程托管给服务器上一个独立的会话，SSH 断开它照样跑。

```bash
tmux new -s exp1          # 新建会话
# 在里面启动训练
# 按 Ctrl+B 再按 D 脱离，可以放心断开 VPN
tmux attach -t exp1       # 下次登录回来接着看
tmux ls                   # 列出所有会话
```

本地如果已经调好了 `.tmux.conf`（鼠标支持、前缀键改成 `Ctrl+A` 之类），直接搬过去，不用重配：

```bash
scp ~/.tmux.conf lab:~/.tmux.conf
```

到服务器上让它生效：

```bash
tmux kill-server 2>/dev/null    # 杀掉旧会话，新配置才会被读取
tmux new -s test                # 验证：鼠标能点窗格，Ctrl+A 前缀生效
```

搬之前值得确认两件事：配置里有没有依赖插件管理器（tpm）或 macOS 专用命令（`pbcopy`、`reattach-to-user-namespace`），有的话在 Linux 上会失效；以及 `tmux -V` 两边版本差距大不大，跨大版本有少量配置项名字变过。

还有一个细节：前缀键改成 `Ctrl+A` 之后，bash 里原本"光标跳到行首"的 `Ctrl+A` 被占用了，想用那个功能要连按两次。

## 十、目录约定

最后是目录结构。核心原则只有一条：**代码、数据、产出三分离**。

```
~/
├── code/                 # git 仓库，一个项目一个文件夹，只放代码
├── data/      -> 大盘     # 数据集，只读，多项目共用
├── outputs/   -> 大盘     # checkpoint、日志、生成结果
│   └── <项目名>/
│       └── 20260914_lr1e-4_bs32/     # 一次实验一个目录，名字带日期和关键超参
└── .cache/    -> 大盘     # HuggingFace、pip、torch 缓存
```

三者的生命周期完全不同：代码进 git，随时可以删掉重新克隆；数据是只读的、体积大、多个项目共用；产出是可再生的、体积最大、最先被清理。混在一个仓库里是新手最常见的灾难——`.git` 里一旦塞进了 checkpoint，整个仓库就废了。所以每个仓库的 `.gitignore` 里要有 `outputs/`、`data/`、`*.pt`、`*.ckpt`、`wandb/`。

家目录通常在一块不大的系统盘上，先用 `df -h` 看清楚哪块盘大、挂在哪，再决定软链到哪里。如果有 `/data` 这类共享盘，问一句管理员"大文件应该放哪个目录"再动手。

每次实验的产出目录里至少留三样东西：完整的配置（超参、命令行）、训练日志、最终 checkpoint，另外把当次代码的 commit hash 记进去。三个月后你一定会忘记某个结果是怎么跑出来的，这几样是唯一的救命稻草。

## 踩过的坑

| 报错原文 | 根因 | 解法 |
|---|---|---|
| `安装器遇到了一个错误，导致安装失败` | 界面提示无信息量，真实原因在 `/var/log/install.log` | 先读日志再动手 |
| `./postinstall: Bootstrap failed: 5: Input/output error` | 第三方卸载工具把 launchd 服务永久标记为 disabled | `sudo launchctl enable system/<service>` 后重装 |
| 「无法移除文件夹」，访达和 `sudo rm` 都删不掉 | `/Library/SystemExtensions/` 受 SIP 保护，设计如此 | 不用删，重装时会被接管 |
| `登陆失败: 用户名或密码错误`（所有组别） | 能弹出组列表说明网关已通，纯账号问题 | 找管理员确认组别与账号开通状态 |
| `Connection closed by ... port 22` | VPN 刚连上路由未就绪（未完全确证） | 等几秒重试 |
| `The authenticity of host ... can't be established` | 不是报错，是首次连接的指纹确认 | 输 `yes` |
| `GnuTLS recv error (-110): The TLS connection was non-properly terminated` | TLS 握手中途被 RST 中断 | SSH 反向端口转发借用本地代理 |

一条贯穿始终的经验：**弹窗上的错误提示基本没有诊断价值，真正的原因都在日志里。** 那个"安装失败，请联系软件生产企业"如果当真去联系厂商，大概永远查不到是被卸载工具禁用了服务。

## 可复用清单

本地 `~/.ssh/config`：

```ini
Host lab
    HostName 203.0.113.10
    User <user>
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 6
    RemoteForward <PROXY_PORT> 127.0.0.1:<PROXY_PORT>
```

服务器端一次性初始化，登录后跑一遍：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. conda 初始化（全局 conda 只读，新环境会落到 ~/.conda/envs）
conda init bash
conda config --set auto_activate_base false

# 2. 国内镜像与缓存目录
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
cat >> ~/.bashrc <<'EOF'
export HF_ENDPOINT=https://hf-mirror.com
export HF_HOME=~/.cache/huggingface
export TORCH_HOME=~/.cache/torch
export PIP_CACHE_DIR=~/.cache/pip
EOF

# 3. git 只对 GitHub 走隧道代理，<PROXY_PORT> 换成本地代理的混合端口
git config --global http.https://github.com.proxy http://127.0.0.1:<PROXY_PORT>

# 4. 目录骨架
mkdir -p ~/code ~/outputs ~/.cache

echo "done; 执行 source ~/.bashrc 生效"
```

日常三条命令：

```bash
ssh lab                                   # VPN 连上后
nvidia-smi                                # 看哪张卡空着
CUDA_VISIBLE_DEVICES=1 python train.py    # 在 tmux 里跑
```

## 未尽事项

- **数据盘还没落位**。`df -h` 的结果还没看，`data/` 和 `outputs/` 的软链目标待定，要先问清楚课题组的约定。
- **代理依赖本地机器**。现在的方案要求笔记本开着、代理运行、VPN 连着。如果课题组本身有代理，应该换过去，那才是长期方案。
- **实验记录还是手工的**。wandb 或 TensorBoard 应该在跑第一个正式实验前接上，靠 `print` 和翻日志找结果撑不过三个月。
- **环境没有固化**。`environment.yml` 要在环境稳定后导出，否则换机器无法复现。

配置这种事的价值不在于配得多漂亮，而在于配完之后就不用再想它。这套东西目前能撑到"打开电脑就能跑实验"，剩下的等真正被卡住时再补。
