---
title: "Building hector.software From Scratch: Domain, Cloudflare, and Hexo"
date: 2026-09-12 12:00:00
lang: en
categories: Meta
tags:
  - Hexo
  - NexT
  - Cloudflare
  - Build Log
---

This is the first post on this blog — a record of how `hector.software` went from an idea to an actual site. I hit a few snags along the way and wrote them down for future-me, and for anyone passing by who's doing the same thing. Anywhere an account, key, or IP address would appear, I've swapped in a placeholder value — none of the numbers below are real.

<!-- more -->

## The domain: an education email + the GitHub Student Developer Pack

The domain came from Name.com's student offer inside the [GitHub Student Developer Pack](https://education.github.com/pack): first verify student status at `education.github.com` with an education email, then authorize a third-party OAuth app called "Name.com Education Pack" on GitHub (read-only access to public profile info, just to prove eligibility), which redirects to Name.com where you register or log into a **separate** Name.com account to actually complete the domain purchase. These two steps are distinct — the GitHub authorization only unlocks the offer; the domain account itself is a standalone Name.com username/password, not tied to your GitHub login.

One tip: if student verification fails, it's usually because your school's email domain isn't on GitHub's recognized list — uploading a student ID for manual review usually works.

## Choosing a server: from "race for a free Oracle box" to giving up on that

The original plan was to copy the stack of a blogger I follow who runs Hexo + NexT, and grab a permanently-free ARM instance on [Oracle Cloud Always Free](https://www.oracle.com/cloud/free/) along the way (currently 2 OCPU / 12GB RAM — a lot of tutorials online still quote the old "4 core / 24GB", which Oracle scaled back in mid-2026).

That plan stalled at payment verification: Oracle's policy explicitly states it does **not accept virtual or prepaid cards**, only real credit or debit cards. I didn't have a real credit card, and the "virtual card" options I looked into (debit cards from newer banks backed by a real balance) had inconsistent success rates — and a rejected card can get an account flagged as high-risk. Rather than sink more time into that, I switched to options that need no card at all:

- **Microsoft Azure for Students**: $100 of credit, zero credit card required, just education-email verification — kept as a fallback for whenever I actually want a real box to tinker with.
- **Cloudflare Pages**: since Hexo only outputs static files anyway, there's no real need for a "server" at all — static hosting is simpler and this is what the blog actually runs on.

## Migrating DNS to Cloudflare

Moving the domain's DNS from Name.com's native DNS to Cloudflare took two steps:

1. In the Cloudflare dashboard, use "Add a domain" (the newer UI renamed this from "Add a site" — easy to miss, it's the card in the middle of the home screen) to add `hector.software`. Cloudflare scans the existing DNS records automatically.
2. Take the two nameservers Cloudflare assigns (something like `ns1.example-cf-ns.com` / `ns2.example-cf-ns.com` — the actual values are account-specific) back to Name.com's nameserver management page and replace the defaults.

The scan turned up a leftover A record pointing at an old VPS from years back (example: `hector.software → 203.0.113.10`) — that box is long gone and the IP has since been reclaimed, so it was a dead record. Deleted it and let Cloudflare Pages take over DNS later.

Nameserver propagation is officially 1-2 hours, up to 24 in the worst case.

## Hexo + NexT

Local setup: Node.js, npm, Git, and the GitHub CLI (already logged in via `gh auth login`).

```bash
mkdir -p ~/Code/hector-blog && cd ~/Code/hector-blog
npm install -g hexo-cli
hexo init .
npm install
```

The theme is [NexT](https://theme-next.js.org/), on the **Pisces** scheme — the two-column layout with a fixed sidebar. NexT ships four schemes (Muse / Mist / Pisces / Gemini), and only Pisces and Gemini have this fixed-sidebar layout; I initially guessed Mist by mistake and the result looked nothing like the reference site.

```bash
npm install hexo-theme-next hexo-generator-searchdb
```

One gotcha worth noting: **don't install the theme with pnpm** (at least not with default settings). `hexo-theme-next` has a few runtime dependencies (`hexo-util`, `js-yaml`, `css`) that are only listed under devDependencies — it's effectively relying on npm's classic flat node_modules to have them lying around nearby. pnpm's default strict, isolated node_modules doesn't do that implicit fallback, so it fails hard with `Cannot find module 'hexo-util'`. The fix is simple: delete `node_modules` and `pnpm-lock.yaml`, and just use plain `npm install`.

All theme customization lives in `_config.next.yml` at the project root (not the copy bundled inside `node_modules`), so upgrading the theme later with `npm update` won't conflict with local changes:

```yaml
scheme: Pisces
creative_commons:
  sidebar: true
language_switcher: true
local_search:
  enable: true
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  archives: /archives/ || fa fa-archive
  talks: /talks/ || fa fa-comments
```

## Deploying: Cloudflare Pages

Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**, pick the GitHub repo holding the blog source, and set the build config:

- Build command: `hexo generate`
- Build output directory: `public`
- An environment variable `NODE_VERSION=20`

After saving, Cloudflare gives you a temporary `*.pages.dev` URL. Once the build succeeds, go to that Pages project's **Custom domains** and add `hector.software` and `www.hector.software` — since the domain is already a Cloudflare zone, this writes the necessary DNS records automatically, no manual DNS editing needed, and TLS certificates are issued and renewed automatically too.

## Wrap-up

End to end, aside from the local project setup, this needed essentially no real "server" at all — a static blog paired with Cloudflare Pages is about as low-maintenance as it gets: no OS updates to babysit, no ports to open, no certificates to renew by hand. If I ever want to run something else (say, hosting inference for a small model), the Azure for Students credit is there as a fallback.

More research and engineering notes to come — thanks for stopping by.
