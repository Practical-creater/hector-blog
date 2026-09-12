# hector.software

Source of [hector.software](https://hector.software) — a static blog built with
[Hexo](https://hexo.io) and the [NexT](https://theme-next.js.org) theme, deployed
by Cloudflare Workers Builds on every push to `main`.

## Daily workflow

```bash
cd ~/Code/hector-blog
git pull                          # pick up anything committed elsewhere
hexo new post "my-post-slug"      # creates source/_posts/my-post-slug.md
hexo server                       # preview at http://localhost:4000 (Ctrl+C to stop)
git add -A && git commit -m "post: my post" && git push
```

About a minute after the push the site is live. Nothing in the Cloudflare
dashboard needs touching for ordinary posts.

## Where things live

| Path | What |
|---|---|
| `source/_posts/` | posts (Markdown + front-matter) |
| `source/_drafts/` | unpublished drafts (`hexo new draft`, `hexo publish`) |
| `source/images/`, `source/fonts/` | static assets, referenced as `/images/...`, `/fonts/...` |
| `source/about/`, `source/talks/` | standalone pages |
| `_config.yml` | site settings (title, url, language, plugins) |
| `_config.next.yml` | theme settings (layout, menu, fonts, features) |
| `source/_data/variables.styl`, `styles.styl` | colour palette and self-hosted `@font-face` rules |
| `source/_data/languages.yml` | custom UI strings (e.g. the `Talks` menu label) |
| `wrangler.jsonc` | Cloudflare deploy config: `public/` is served as static assets |

## Build & deploy

- Cloudflare runs `npm ci` → `npm run build` → `npx --yes wrangler deploy`.
- `wrangler` is deliberately **not** a dependency: its per-platform optional
  binaries make `package-lock.json` incompatible between npm 10 and 11.
- Use `npm`, not `pnpm`: the NexT theme relies on hoisted (undeclared)
  dependencies and fails to load under pnpm's isolated `node_modules`.
- `.github/workflows/build.yml` builds every pull request so dependency
  bumps are verified before merge.
- Node version is pinned in `.nvmrc` (matches the Cloudflare build image).

## Setting up on a new machine

```bash
git clone git@github.com:Practical-creater/hector-blog.git ~/Code/hector-blog
cd ~/Code/hector-blog && npm install
npx hexo server
```
