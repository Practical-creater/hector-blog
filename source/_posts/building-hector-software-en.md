---
title: "Building hector.software From Scratch: Domain, Cloudflare, Hexo, and Every Pitfall Along the Way"
date: 2026-09-12 12:00:00
categories: Meta
tags:
  - Hexo
  - NexT
  - Cloudflare
  - Build Log
---

This is the first post on this blog, and it documents how `hector.software` went from a bare domain name to a site you can actually open. I've written it in the order things really happened, including three failed builds and the verbatim error messages from each, so that someone who has never set up a website can follow along and understand not just what to type but why. Anywhere an account ID, IP address, or key would appear, I've substituted an example value — none of the numbers below are real.

<!-- more -->

## First, the parts a blog is made of

It's worth two minutes up front to sort out a few terms, otherwise every step later feels arbitrary.

A **domain** is your street address — `hector.software`. **DNS** is the directory service that translates that address into "where the content actually lives." **Hexo** is a printing press: you write posts in Markdown and it typesets them into the HTML files a browser can display. **Cloudflare** is the bookshop where the printed pages sit, and where readers from anywhere come to pick them up. The **GitHub repository** is the drawer holding the manuscripts — every source file of the blog (posts, configuration, theme settings) lives there.

Chained together, the workflow is this: I write a post locally and push it into the GitHub drawer; Cloudflare notices the drawer changed, runs the press, and puts the freshly printed HTML on the shelf; a reader types the domain, and DNS walks them to the shop. Once that chain is clear, every configuration step below is just answering one question — how does this link connect to the next one?

## The domain: the first thing a school email saves you

The domain came from Name.com's offer inside the [GitHub Student Developer Pack](https://education.github.com/pack), free for the first year. You verify student status at `education.github.com` with a school email, then click the Name.com offer in the pack. GitHub asks you to authorize a third-party app called "Name.com Education Pack" — it reads only your public profile, and its only job is to prove to Name.com that you're a verified student. After that you're redirected to Name.com, pick a TLD that's covered by the offer (`.software`, `.dev`, `.app` are all on the list), and at checkout Name.com asks you to **register a separate Name.com account**.

That last part is where I tripped up months later. The Name.com account and the GitHub account are two unrelated things. GitHub authorization only unlocks the coupon; the domain itself sits under a Name.com username and password. When I came back to change DNS I couldn't remember that password, assumed I had "logged in with GitHub," and wasted time digging through GitHub's authorization log. The real recovery route is "Forgot Username" on Name.com's login page — it asks for your **domain name**, not an email, and mails the username to the contact address on file; with the username you then use "Forgot password" for a reset link. WHOIS privacy on the domain doesn't interfere; that only hides your details from public lookups, Name.com itself still knows where to send mail.

Plain advice: the moment you register a domain, put the username, password, and the email you used into a password manager. The domain is the root of everything; losing access to it costs more than anything else here.

## Servers: why I ended up not buying one

The original plan was to copy the stack of a blogger I follow — Hexo with the NexT theme, running on a server of my own — and to grab the permanently free ARM machine from [Oracle Cloud Always Free](https://www.oracle.com/cloud/free/) on the way. Its current allowance is 2 OCPU and 12 GB of memory (a lot of tutorials still quote "4 cores / 24 GB"; that was cut in mid-2026).

That route stalled at payment verification. Oracle asks for a card to verify identity at signup and states explicitly that it does **not accept virtual or prepaid cards**, only real credit or debit cards. I don't have a credit card, and the "virtual card" workarounds I looked into came down to luck — and if a card gets flagged as high-risk, the whole account can be restricted, sometimes unrecoverably. Not worth the time.

Stepping back, there was no actual need for a server. Hexo produces static files — a pile of HTML, CSS, and JavaScript, with no database and no backend logic. The natural home for static files isn't a virtual machine you patch, open ports on, and renew certificates for; it's a static hosting service that you hand the files to and let distribute them. Cloudflare offers exactly that, free, with HTTPS and a global CDN included. And if I ever do want a machine to tinker with, [Azure for Students](https://azure.microsoft.com/en-us/free/students) gives students $100 of credit with no credit card at all — a fine fallback.

## Handing DNS to Cloudflare

If Cloudflare is going to host the site, it needs to control the domain's DNS. That means switching the domain's nameservers from Name.com to Cloudflare. The registrar stays Name.com; only the "directory service" changes.

In the Cloudflare dashboard, type the domain into the "Add a domain" card in the middle of the home screen. Older tutorials call this "Add a site" — the new UI renamed it and I hunted for a while. After choosing the free plan, Cloudflare scans the domain's existing records and asks you to confirm them. Mine had two: an A record pointing the root domain at `203.0.113.10` (example value), and a CNAME pointing `www` back at the root. That IP belonged to a cloud VM I'd shut down years ago — the machine is gone and the provider has reclaimed the address, so the record pointed at nothing. I deleted both, because Cloudflare will write the correct records itself when the custom domain gets attached later, and stale ones would only conflict.

After confirming, Cloudflare hands you two nameservers, something like `ada.ns.cloudflare.com` and `bob.ns.cloudflare.com` (every account gets a different pair). Back on Name.com's nameserver management page, replace the default `ns1.name.com` entries with those two and save. Propagation takes one to two hours officially, up to 24 in the worst case, and until it completes the domain overview in Cloudflare shows "Waiting for your registrar to propagate your new nameservers." That page also has a prominent "Create Worker" button on the right; it's unrelated to the next step and can be ignored — our project gets created from a different entry point.

## Getting the blog running locally: Hexo + NexT

Locally you need Node.js, npm, and Git; the GitHub CLI (`gh`) saves a lot of clicking too. The project lives in `~/Code/hector-blog` — not on the desktop, which turns into a junkyard after a few projects.

```bash
mkdir -p ~/Code/hector-blog && cd ~/Code/hector-blog
npm install -g hexo-cli
hexo init .
npm install
```

`hexo init` pulls down an official starter with `_config.yml`, `source/_posts/` (where posts go), and `scaffolds/` (templates for new posts) already in place. The theme is [NexT](https://theme-next.js.org/), installed with npm:

```bash
npm install hexo-theme-next hexo-generator-searchdb
```

NexT ships four layout schemes: Muse, Mist, Pisces, and Gemini. The first two are single-column, with a sidebar that stays hidden until you tap the button in the bottom-left corner; the last two are two-column, with the sidebar fixed in place. The site I was modelling says "Powered by Hexo & NexT.Mist" in its footer, and the dark panel with the avatar is simply **Mist**'s slide-in sidebar. Working from a screenshot taken with that sidebar open, I mistook it for Pisces' two-column layout, switched, got a page with a completely different structure, and switched back. The lesson is trivial: if you want to borrow a site's layout, read the scheme name in its footer instead of guessing from a picture.

All theme customization goes into a file called `_config.next.yml` at the project root, rather than editing the copy of the theme's config inside `node_modules`. That way upgrading the theme later never overwrites your changes:

```yaml
scheme: Mist
creative_commons:
  sidebar: true
local_search:
  enable: true
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  archives: /archives/ || fa fa-archive
  talks: /talks/ || fa fa-comments
```

A few fields in the root `_config.yml` must change: `title` and `author` to your own; `url` to `https://hector.software`, which Hexo uses to build every internal link, the RSS feed, and the sitemap — get it wrong and every link points at `example.com`; and `language: en`, so that navigation, dates and labels like "Read more" are all English while individual posts can be written in either language — the same arrangement as the site I was modelling. I first tried `[zh-CN, en]` with a `lang:` tag on every post so the interface would follow each post's language; the result was a site that flipped languages page by page, plus a footer language dropdown that led to 404s, so I settled on an English interface throughout.

Run `hexo server` and open `http://localhost:4000` to see the result locally; edits to config or posts reload automatically.

**The first pitfall is here.** I originally installed the theme with pnpm, and `hexo` immediately failed with:

```
ERROR Script load failed: node_modules/hexo-theme-next/scripts/filters/post.js
Error: Cannot find module 'hexo-util'
```

A look at `hexo-theme-next`'s `package.json` explains it: at runtime the theme requires `hexo-util`, `js-yaml`, and `css`, but `js-yaml` is only listed under devDependencies and `hexo-util` and `css` aren't listed at all. It works on most machines because npm's `node_modules` is flat — those packages get hoisted to the top level as dependencies of other dependencies, and the theme can `require` them "by accident." pnpm's `node_modules` is strictly isolated; an undeclared dependency simply isn't there. This has a name, phantom dependencies, and it isn't pnpm's fault, but as a user the cheapest fix is not to fight the theme: delete `node_modules` and `pnpm-lock.yaml` and use plain `npm install`. The whole Hexo ecosystem assumes npm anyway.

## Deploying: Pages has become Workers

Once the site works locally, push the source to GitHub. With `gh`, creating the repository and pushing is one command:

```bash
git init
git add -A
git commit -m "Init Hexo + NexT blog"
gh repo create hector-blog --public --source=. --remote=origin --push
```

The starter's `.gitignore` already excludes `node_modules/` and `public/`. One is dependencies, the other is build output; neither belongs in the repository — dependencies can be reinstalled with `npm install` at any time, and the output gets regenerated by the deployment environment. The repository holds only the manuscript.

Back in the Cloudflare dashboard, open **Workers & Pages** in the left menu, click Create, and choose to import from a Git repository. This is the point where GitHub enters the picture, so it's worth explaining why. Cloudflare doesn't store your source code; it's merely authorized to read your GitHub repository. That authorization takes the form of a Cloudflare app installed on your GitHub account, with you choosing which repositories it may see. From then on, every push to the repository's main branch makes GitHub notify Cloudflare, which clones the repository onto a clean build machine, installs dependencies, runs the build, and publishes the result. That's what continuous deployment means. The alternative — running the deploy command by hand from your own laptop every time — works too, but it means a different or broken laptop can't update the site, and the deployments leave no record. Letting GitHub be the single source of truth and letting Cloudflare watch it is the more restful arrangement.

After choosing the repository you land on a page titled "Set up your application — Configure your Worker project." Note: **Worker**. Older tutorials describe Cloudflare Pages with a dedicated form containing a "build output directory" field and environment variables; new projects now all go through the Workers flow, that form no longer exists, and in its place are two commands plus a config file:

- **Build command**: `npm run build`.
- **Deploy command**: leave the default, `npx wrangler deploy`.

Wrangler is Cloudflare's command-line tool; `wrangler deploy` publishes things. It needs to know *what* to publish, and that comes from `wrangler.jsonc` in the repository root:

```jsonc
{
  "name": "hector-blog",
  "compatibility_date": "2026-09-01",
  "assets": {
    "directory": "./public",
    "not_found_handling": "404-page"
  }
}
```

`assets.directory` points at Hexo's output folder `public`; that one line is the replacement for the old "build output directory" box. `name` must match the Project name in the wizard. The page also offers "Build variables" — environment variables for the build process — which this blog doesn't need.

## Three failed builds, and the reason for each

After clicking Deploy, the build log splits into five stages: Initializing, Cloning, Installing, Building, Deploying. I fell over once in Installing and twice in Building. The errors are worth recording verbatim, because both are the kind you'll meet again on other projects.

### First: npm error Invalid Version

The Installing stage failed outright, with only one useful line in the log:

```
Detected the following tools from environment: npm@10.9.2, nodejs@24.18.0
Installing project dependencies: npm clean-install --progress=false
npm error Invalid Version:
```

`Invalid Version` followed by nothing — no clue at all. Running the same command, `npm ci`, locally worked fine. The difference was versions: my local npm was 11.6.2, the build machine's was 10.9.2.

The background: earlier, "for reproducibility," I had added `wrangler` to the project's devDependencies. Wrangler depends on a runtime called workerd, which is published as one binary package per operating system — `@cloudflare/workerd-darwin-64`, `@cloudflare/workerd-linux-64`, and so on — declared as optional dependencies so that only the one matching the current OS gets installed. The trouble is that npm 10 and npm 11 record these per-platform optionals in `package-lock.json` differently. A lockfile written by npm 11 is unreadable to npm 10, hence `Invalid Version`; and when I regenerated the lockfile with npm 10.9.2, my local npm 11 refused it in turn:

```
npm error `npm ci` can only install packages when your package.json and
package-lock.json are in sync.
npm error Missing: @cloudflare/workerd-darwin-64@ from lock file
npm error Missing: @cloudflare/workerd-linux-64@ from lock file
```

Mutually incompatible; fixing one side breaks the other. A quick aside on `npm ci` versus `npm install`: `install` is forgiving — if the lockfile and `package.json` disagree, it quietly updates the lockfile. `ci` is meant for automated environments and demands that the lockfile match `package.json` exactly, refusing to install otherwise, so that every build installs precisely the same thing. Build machines use `ci`, which is why any format difference in the lockfile is fatal.

The fix removes the cause entirely: `wrangler` never needed to be in `package.json`. The deploy command is `npx wrangler deploy`, and `npx`'s whole behavior is "if it isn't installed locally, download a copy and run it" — the build machine fetches a wrangler matching its own OS at deploy time. Drop it from the dependencies and the lockfile no longer contains any per-platform binaries, and both npm versions read it happily.

### Second and third: /bin/sh: 1: hexo: not found

With the lockfile fixed, Installing passed and Building fell over:

```
Executing user build command: hexo generate
/bin/sh: 1: hexo: not found
Failed: error occurred while running build command
```

The shell can't find a command named `hexo`. Locally, `hexo` works from any directory because the very first `npm install -g hexo-cli` put it in a global bin directory that's on the PATH. The build machine is a clean box that installed only the project's dependencies. The `hexo` command *does* exist there — at `node_modules/.bin/hexo` inside the project — but Cloudflare runs the Build command by handing the literal string to `/bin/sh`, and `node_modules/.bin` isn't on sh's PATH.

My first reflex was to add `hexo-cli` to devDependencies and retry. Identical failure. Only afterwards, reading `node_modules/hexo/package.json`, did I see that the `hexo` core package declares `"bin": {"hexo": "./bin/hexo"}` itself and already depends on `hexo-cli` — so `.bin/hexo` had been present since the very first build. The problem was PATH, start to finish.

The correct fix is to change the Build command from `hexo generate` to `npm run build`. The starter's `package.json` already contains `"build": "hexo generate"`, and `npm run` prepends `node_modules/.bin` to PATH before executing any script. That's npm's own behavior, independent of which machine it runs on, so it works identically locally and on the build machine. `npx hexo generate` would also work, since `npx` looks in the local `node_modules` first. The field is edited under the project's Settings; afterwards, hit Retry build on the latest build, or push any new commit to trigger one.

### A word on the allow-scripts warnings

The log also carried a yellow block:

```
npm warn allow-scripts 4 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   hexo-util@4.0.0 (postinstall: npm run build:highlight)
```

This means npm no longer runs third-party dependencies' install scripts by default unless you explicitly approve them — a supply-chain security measure, not an error. `hexo-util`'s postinstall script generates an alias table for code highlighting, and that table is already shipped inside the published package, so whether the script runs makes no difference. Safe to ignore.

## Attaching the domain

With all three problems fixed, all five stages turn green and Cloudflare gives you a temporary address of the form `hector-blog.<account>.workers.dev`; if the blog opens there, publishing works. Then go to the project's Overview, find "Domains and routes" on the right, click Custom domains, and add `hector.software` and `www.hector.software` one after the other. Because the domain's DNS is already in Cloudflare's hands, it writes the records and issues the HTTPS certificate itself, and the domain opens within minutes. The two stale records deleted earlier get replaced by correct ones at this step.

The precondition is that nameserver propagation has finished. If the domain overview still says "Waiting for your registrar," the binding may ask you to wait — come back a little later.

## Writing and maintaining from here on

The daily writing loop is short. In the project directory:

```bash
hexo new post "my-new-post"      # creates source/_posts/my-new-post.md
hexo server                       # local preview at http://localhost:4000
git add -A && git commit -m "post: my new post" && git push
```

About a minute after the push, a new build appears in Cloudflare's Deployments tab, and when it finishes the site is updated. The Chinese and English versions of a post are two separate files; write the body in whichever language you like — the interface stays English. To write without publishing, `hexo new draft "title"` puts the file in `source/_drafts/`, `hexo server --drafts` previews it, and `hexo publish "title"` moves it to the posts folder when it's ready.

A few things you never touch again: the build settings in the Cloudflare dashboard only change if the commands change; `node_modules` and `public` never enter the repository; theme upgrades are `npm update hexo-theme-next`, and your customizations in `_config.next.yml` survive them. On a new machine, `git clone` plus `npm install` restores the whole environment — which also means the repository *is* the backup, and there's no server to back up separately.

When a build fails, read the Deployments log first and find which of the five stages broke. An Installing failure is usually the lockfile; a Building failure, suspect the command and PATH first; a Deploying failure, check `wrangler.jsonc`.

## Wrap-up

What actually took time here wasn't "configuration" but understanding what each layer does: the registrar and the DNS provider can be different companies, GitHub and Cloudflare handle source and publishing respectively, and Hexo is just a tool that turns Markdown into HTML. None of the three build failures was deep either — one was two npm versions disagreeing about the same lockfile, two were a command missing from the build machine's PATH. Once the cause is clear, each fix is a line or two.

More research and engineering notes to come. Thanks for stopping by.
