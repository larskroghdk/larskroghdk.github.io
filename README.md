# larskroghdk.github.io

Source for [larskroghdk.github.io](https://larskroghdk.github.io/): a static blog built with
[Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
theme, deployed to GitHub Pages by GitHub Actions.

## Architecture

```text
content/*.md ──► Hugo Extended + PaperMod (Hugo Module) ──► public/ ──► GitHub Pages
                         ▲
        .github/hugo-version.env (pinned Hugo release + SHA-256)
        go.mod / go.sum         (pinned PaperMod commit)
```

| Path | Purpose |
| --- | --- |
| `hugo.toml` | Site configuration: PaperMod parameters, menu, outputs (RSS, JSON search index), markup |
| `go.mod`, `go.sum` | Hugo Module definition. Pins PaperMod to an exact commit with a checksum |
| `content/posts/` | Articles. A post is either `name.md` or a page bundle `name/index.md` with its images |
| `content/search.md`, `content/archives.md` | PaperMod search (Fuse.js) and archive pages |
| `archetypes/posts.md` | Front matter template for `hugo new content posts/...` |
| `layouts/_partials/extend_post_content.html` | Related posts under each article (PaperMod extension point) |
| `assets/css/extended/` | The only custom CSS, picked up by PaperMod automatically |
| `static/` | Files copied as-is (favicons) |
| `.github/hugo-version.env` | The Hugo Extended version used by CI, and its checksum |
| `.github/workflows/hugo.yml` | Lint, build, link check, and deploy |
| `.github/workflows/update-hugo.yml` | Monthly pull request that updates Hugo and PaperMod |
| `.github/dependabot.yml` | Weekly pull requests for GitHub Actions updates |
| `.markdownlint-cli2.yaml`, `.htmltest.yml` | Markdown lint and internal link check configuration |

### CI/CD

`Build and deploy` (`hugo.yml`) runs on every pull request, every push to `main`, and on demand:

| Job | What it does |
| --- | --- |
| Lint | markdownlint on all Markdown, actionlint on all workflows |
| Build | Installs the pinned Hugo Extended (checksum verified), verifies module checksums, runs a production build that fails on any warning, then checks every internal link and asset with htmltest |
| Deploy | Only for `main`: publishes the build with the official Pages actions |

The deploy job needs both Lint and Build to pass. Pull requests, including Dependabot and update PRs, never deploy.
Actions are pinned to commit SHAs. The workflow token is read-only except for `pages: write` and `id-token: write` in the deploy job.

## Local development

### Prerequisites

| Tool | Why | Version |
| --- | --- | --- |
| Hugo **Extended** | Builds the site | The version in [`.github/hugo-version.env`](.github/hugo-version.env) (0.158.0 minimum) |
| Go | Hugo Modules download PaperMod through Go | 1.24 or later |
| Git | Hugo Modules and `.Lastmod` from git history | any recent |
| Node.js (optional) | Run markdownlint locally | LTS |

#### Windows (PowerShell)

```powershell
winget install --id GoLang.Go
winget install --id Git.Git
winget install --id Hugo.Hugo.Extended
hugo version   # must say "+extended" and match .github/hugo-version.env
```

To install the exact pinned version instead, download `hugo_extended_<version>_windows-amd64.zip` from the
[Hugo releases](https://github.com/gohugoio/hugo/releases), check it against `hugo_<version>_checksums.txt`
with `Get-FileHash`, and put `hugo.exe` on your `PATH`.

#### Linux (bash)

```bash
# Go and Git from your distribution, for example: sudo apt install golang git
source .github/hugo-version.env
curl -fsSLO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
echo "${HUGO_SHA256}  hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" | sha256sum --check
sudo tar -xzf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" -C /usr/local/bin hugo
hugo version
```

### Run the site

```bash
git clone https://github.com/larskroghdk/larskroghdk.github.io.git
cd larskroghdk.github.io
hugo server                         # http://localhost:1313/ , reloads on save
hugo server --buildDrafts --buildFuture   # include drafts and scheduled posts
```

The first run downloads PaperMod into Hugo's module cache. Nothing is vendored into the repository.

### Run the CI checks locally

```bash
hugo build --gc --minify --panicOnWarning --printPathWarnings --printI18nWarnings
hugo build --gc --baseURL / --destination public-linkcheck --quiet
go run github.com/wjdp/htmltest@v0.17.0
npx markdownlint-cli2
go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.12
```

In Git Bash on Windows, prefix the link-check build with `MSYS_NO_PATHCONV=1`, otherwise Git Bash rewrites
`--baseURL /` into a Windows path. PowerShell doesn't need it.

## Writing posts

### Create a post

```bash
hugo new content posts/my-post/index.md   # page bundle: keep images next to index.md
hugo new content posts/my-post.md         # single file, no images
```

The archetype creates this front matter:

```yaml
---
title: "My Post"
description: ""
date: 2026-11-19T10:00:00+01:00
draft: true
tags: []
categories: []
---
```

- **Summary**: the text before `<!--more-->` is the summary in lists, search and RSS. Without it, Hugo uses the first paragraph(s).
- **Images**: put them in the bundle and reference them relatively: `![Alt text](diagram.png "Caption")`.
  For a cover and Open Graph image, add `cover.image: "diagram.png"`, `cover.alt` and `cover.relative: true`.
- **Internal links**: use `[text]({{< relref "posts/other-post" >}})`. A broken `relref` fails the build.
- **Code**: fenced blocks with a language (`powershell`, `go`, `yaml`, `json`, `hcl`, `bash`) get syntax highlighting and a copy button.
- **Related posts** are picked by shared tags and keywords. Use existing tags where they fit: see `/tags/`.
- A table of contents is shown on every post. Turn it off for one post with `ShowToc: false`.

### Drafts and scheduled posts

- `draft: true` posts are only built with `--buildDrafts`. They are never published.
- A post with a future `date` is published by the first build after that time. Hugo has no clock of its own, so
  the post appears with the next push or a manual run of **Build and deploy** (Actions → Build and deploy → Run workflow).
- `expiryDate` removes a post from the site after that date, on the next build.

### Publish

```bash
git switch -c post/my-post
git add content/posts/my-post
git commit -m "Add post: My post"
git push -u origin post/my-post
gh pr create --fill
```

When the checks pass, merge the pull request. The push to `main` deploys the site.
Pushing straight to `main` also deploys, if the branch rules allow it.

## GitHub configuration

These settings can't be stored in the repository. Set them once:

1. **Settings → Pages → Build and deployment → Source: GitHub Actions.**
2. **Settings → Actions → General → Workflow permissions**: keep *Read repository contents* as the default, and tick
   **Allow GitHub Actions to create and approve pull requests**. The update workflow needs this to open its pull request.
3. **Settings → Rules → Rulesets → New branch ruleset** for `main`: require a pull request, and require the status
   checks **Lint** and **Build**. This is what stops untested dependency updates from reaching `main`, and therefore Pages.
4. **Settings → Environments → github-pages**: created by the first deployment. Keep the deployment branch rule limited to `main`.
5. **Settings → Code security**: enable Dependabot alerts and security updates. Version updates are already configured in `.github/dependabot.yml`.

### Custom domain

1. Create DNS records with your DNS provider:
   - apex domain (`example.com`): `A` records to `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
     (and optionally `AAAA` to `2606:50c0:8000::153`, `2606:50c0:8001::153`, `2606:50c0:8002::153`, `2606:50c0:8003::153`),
   - subdomain (`blog.example.com`): `CNAME` to `larskroghdk.github.io`.
2. **Settings → Pages → Custom domain**: enter the domain, wait for the DNS check, then tick **Enforce HTTPS**.
3. Optionally verify the domain for your account (**Profile settings → Pages**) to prevent takeover.
4. Change `baseURL` in `hugo.toml` to the new domain so local builds and the link checker match.

No `CNAME` file is needed: with Actions-based deployments, GitHub ignores it and uses the setting.
The deploy build takes its base URL from `actions/configure-pages`, so links are correct as soon as the domain is active.
The same mechanism handles project sites served from `https://<user>.github.io/<repo>/`.

## Updating Hugo and PaperMod

| Dependency | Pinned in | Updated by |
| --- | --- | --- |
| Hugo Extended | `.github/hugo-version.env` (version and SHA-256) | `Update Hugo and PaperMod`, monthly or on demand |
| PaperMod | `go.mod` / `go.sum` | `Update Hugo and PaperMod`, monthly or on demand |
| GitHub Actions | commit SHAs in the workflows | Dependabot, weekly |

The update workflow pushes `automation/update-hugo`, opens a pull request, and starts **Build and deploy** on that
branch. Pull requests opened with `GITHUB_TOKEN` don't trigger `pull_request` workflows, so it dispatches the run
itself; the result shows as checks on the pull request. Read the
[Hugo release notes](https://github.com/gohugoio/hugo/releases) before merging. Preview locally with:

```bash
gh pr checkout <number>
hugo server
```

To update by hand instead:

```bash
hugo mod get -u github.com/adityatelange/hugo-PaperMod   # PaperMod: updates go.mod and go.sum
# Hugo: edit HUGO_VERSION and HUGO_SHA256 in .github/hugo-version.env,
# taking the checksum of hugo_extended_<version>_linux-amd64.tar.gz from hugo_<version>_checksums.txt
```

PaperMod publishes no semantic version tags that Go modules accept, so the module follows its `master` branch at a
pinned commit. `go.sum` makes every build use exactly that commit.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `this feature is not available in your current Hugo version` or SCSS errors | You are running standard Hugo. Install the **Extended** edition. |
| `failed to download modules` / `go: command not found` | Hugo Modules need Go and Git on `PATH`. Check `go version` and `git --version`. Behind a proxy, set `HTTPS_PROXY`. |
| Build fails with `WARN ... ERROR` | CI runs with `--panicOnWarning`. Run the same build locally, fix the warning (often a deprecated option after a Hugo update). |
| `REF_NOT_FOUND` | A `relref` points to a page that doesn't exist. Use the path below `content/`, without extension. |
| htmltest: `target does not exist` | A link or image points to a missing file. Check relative image paths in page bundles. |
| markdownlint errors | Run `npx markdownlint-cli2` locally. Rules are in `.markdownlint-cli2.yaml`. |
| `sha256sum: WARNING: 1 computed checksum did NOT match` | `HUGO_SHA256` doesn't match `HUGO_VERSION`. Take the checksum for the linux-amd64 Extended tarball from the release's checksum file. |
| Deploy job: `Get Pages site failed` / `HttpError: Not Found` | Pages isn't enabled with source **GitHub Actions** (Settings → Pages). |
| Deploy job: `Branch "x" is not allowed to deploy to github-pages` | Only `main` deploys. Merge to `main` or adjust the environment's branch rule. |
| Update workflow: `GitHub Actions is not permitted to create or approve pull requests` | Enable it under Settings → Actions → General → Workflow permissions. |
| A post doesn't appear | It is `draft: true`, its `date` is in the future (build with `--buildFuture` to check), or its `expiryDate` has passed. |
| Site looks unstyled on a custom domain or project site | `baseURL` mismatch. CI sets it from Pages; locally, `hugo server` overrides it. Check the deployed HTML's `<link rel=canonical>`. |
| Old content after deploy | Pages serves through a CDN. Wait a few minutes or hard-refresh. |
