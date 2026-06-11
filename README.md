# jaalso.github.io

Personal security write-up site — custom Jekyll, terminal theme.
Live at **https://jaalso.github.io** once deployed.

---

## Get it live (one-time)

1. Create a **public** GitHub repo named exactly `jaalso.github.io`.
2. Push these files to the `main` branch:

   ```bash
   cd jaalso.github.io
   git init
   git add .
   git commit -m "init: terminal write-up site"
   git branch -M main
   git remote add origin git@github.com:jaalso/jaalso.github.io.git
   git push -u origin main
   ```

3. On GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch**,
   branch `main` / root. Wait ~1 minute; the site builds automatically.

That's it — every future `git push` rebuilds and redeploys.

---

## Run it locally (optional but recommended)

You need Ruby. On Kali:

```bash
sudo apt update && sudo apt install -y ruby-full build-essential zlib1g-dev
gem install bundler
cd jaalso.github.io
bundle install
bundle exec jekyll serve --livereload
```

Open http://127.0.0.1:4000 — edits reload live.

---

## Add a write-up

**As a full post (shows on the site):**
copy `_drafts/_TEMPLATE.md` into `_posts/` and rename it
`YYYY-MM-DD-some-title.md`. The date and tags in the front-matter drive the
homepage, the `/writeups/` list, the `/tags/` index, and the RSS feed — all
automatically. Put images in `assets/img/` and reference them like
`![alt](/assets/img/foo.png)`.

**As a linked repo (just points out):**
add an entry to `_data/external.yml`.

`featured: true` on a post pins a FEATURED badge on the homepage.

---

## Customize

- Colors / fonts: `assets/css/style.css` (`:root` variables at the top).
- Name, links, HTB/THM usernames: `_config.yml`.
- About text: `about.md`.
- Nav links: `_includes/header.html`.

## Use your own domain (later)

You own `bsociety.ch` at Hostpoint. To serve the site from e.g.
`blog.bsociety.ch`: add a `CNAME` file containing `blog.bsociety.ch`, then at
Hostpoint add a DNS `CNAME` record `blog` → `jaalso.github.io`. Enable
**Enforce HTTPS** in Settings → Pages.

## Structure

```
_config.yml         site config + your links
Gemfile             pins the github-pages gem (matches live build)
index.html          home (hero + recent posts)
writeups.html       all posts + linked repos, with live filter
tags.html           auto tag index (built-in site.tags)
about.md            about page
_posts/             your write-ups (Markdown)
_drafts/_TEMPLATE.md  copy this to start a new post
_data/external.yml  write-ups that link out to repos
_layouts/           default / post / page
_includes/          head / header / footer
assets/css/         the terminal theme
assets/img/         screenshots
```
