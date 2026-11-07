# specian.de

Personal site and blog. Static HTML built with [Zola](https://www.getzola.org/) 0.23 (Tera 2 templates, no theme, no JS).

## Write

```sh
zola serve --drafts          # http://127.0.0.1:1111, live reload
```

New post: `content/blog/<slug>/index.md` (colocate images next to it):

```toml
+++
title = "…"
description = "One sentence. Used in lists, feeds and link previews."
date = 2026-11-07
draft = true                 # flip to false to publish
[taxonomies]
tags = ["homelab"]
[extra]
toc = true                   # optional table of contents
+++
```

Things to know:
- Tera syntax in Markdown is evaluated. Wrap literal `{{ }}` / `{% %}` in `{% raw %}…{% endraw %}`.
- The production CSP blocks inline `<script>` and `style=""`. CI fails the build if any appear.
- Code-block languages must be known to Giallo (`error_on_missing_language = true`).

## Deploy

Push to `main`. GitHub Actions builds, then rsyncs `public/` to the Hetzner box as the `deploy` user. The SSH key is forced to `rrsync /var/www/specian.de`, so it can't do anything else.

The host side (nginx vhost, TLS, deploy user) lives in `infra-lab`: `terraform/src/hetzner/specian_site.tf`. DNS lives in `terraform/src/specian/route53-specian.de.tf`.

| URL | Behavior |
|---|---|
| `https://specian.de/` | site |
| `https://specian.de/blog/` | posts |
| `https://www.specian.de/*` | 301 → `https://specian.de/*` |
| `https://blog.specian.de/*` | 301 → `https://specian.de/blog/*` |

Secret required in the repo: `DEPLOY_SSH_KEY` (private half of `~/.ssh/specian_site_deploy`).
