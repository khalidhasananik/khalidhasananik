# How I Built My GitHub Profile README

A record of how `README.md` in this repo (`khalidhasananik/khalidhasananik`) was
built, so it can be rebuilt, redone for someone else, or handed to an AI coding
agent as a working playbook. Every snippet below was actually run and verified
against live GitHub Actions and live image endpoints on 2026-09-20 — this is
not theoretical.

**For humans:** read top to bottom, or jump to "Quick-start checklist" to redo
this fast.

**For AI agents:** this file is written as an executable spec. Section 2
(principles) governs every decision below it — when in doubt, follow those
rules over convenience. Section 8 (lessons learned) documents real bugs hit
while building this; re-read it before touching any of the same widgets, since
the fixes are non-obvious and re-discovering them wastes a lot of round trips.

---

## 1. What this produces

A special GitHub repo named exactly `<username>/<username>` (e.g.
`khalidhasananik/khalidhasananik`) renders its `README.md` on the user's
public profile page (`github.com/<username>`). This doc builds one that is:

- Minimal and professional (not a "recruiter brief," not badge-spam)
- Built from **real** data pulled from the GitHub API and the person's own
  website — never invented
- Composed of widgets that were individually verified to actually render,
  with known-broken alternatives called out so they aren't reintroduced

Final structure, top to bottom:

1. Profile image (full width)
2. Tagline (role/expertise) + location/company
3. Website / LinkedIn / GitHub-follow badges
4. One short bio paragraph
5. Tech stack icons
6. GitHub streak stats card
7. Self-hosted GitHub stats visualization (overview + languages cards)
8. Snake contribution graph
9. 3D contribution graph
10. A static quote
11. Footer link row

---

## 2. Principles (apply these before adding anything)

1. **No invented facts.** Pull real bio/role/company/links from the GitHub
   API (`api.github.com/users/<username>`) and the person's own site. If a
   piece of information can't be verified, leave it out or ask — don't guess
   a job title, employer, or stack.
2. **No dead links, ever.** Before putting any URL (badge, image, org link)
   into the README, check it resolves. A broken image is worse than no
   image.
3. **Prefer things that don't depend on a third party's uptime.** Several
   popular "awesome README" services are unreliable (see §8). Where a
   self-hosted GitHub Actions alternative exists, prefer it — but only if it
   can be made to actually work end-to-end.
4. **Verify with `curl`, not vibes.** After adding or changing any image URL,
   `curl -s -o /dev/null -w "%{http_code}"` it before calling it done.
5. **Every push to a repo with Action-generated content needs a
   pull-rebase.** See §8.6 — this bit us repeatedly.
6. **Keep it minimal.** Don't add a widget just because it exists. Every
   addition in this build was something the user explicitly asked for after
   seeing a reference example.

---

## 3. Prerequisites

- A repo named `<username>/<username>`, public, cloned locally with push
  access (SSH remote in this build: `git@github.com:<username>/<username>.git`)
- A square-ish logo/photo file committed to the repo root (this build used
  `Khalidhasananik.jpg`)
- `git` and `curl` available locally

---

## 4. Step 1 — Gather real data

Don't skip this. Pull the person's actual public profile before writing any
bio copy:

```bash
curl -s "https://api.github.com/users/<username>"
# -> name, bio, company, location, blog, public_repos, followers

curl -s "https://api.github.com/users/<username>/repos?sort=updated&per_page=100"
# -> real recent repos: use these to infer actual current tech stack,
#    not whatever old coursework repos show up in a default GPRM/GitSkins
#    generated README.
```

If the person has a personal website, fetch it too (their own site is
usually more current than their GitHub bio):

```
WebFetch: "https://<their-site>" — extract role, tagline, bio, tech stack,
services, and contact/social links, quoting the actual text.
```

In this build, the GitHub API said the bio was still `"Imma write it later"`
(literally unset), but the personal site had a real, current title
("Project Coordinator", after "AI Ops & Automation Engineer"), employer, and
tech stack. The website won that conflict — it's more current.

**Verify every org/company link before using it**, e.g.:

```bash
curl -s -o /dev/null -w "%{http_code}" "https://api.github.com/orgs/<company-name>"
# 404 here means: don't link it, just use plain text
```

---

## 5. Step 2 — Base skeleton

```html
<div align="center">
  <img src="<image-file>.jpg" width="100%" alt="<Name>" />
</div>

<h3 align="center"><role/expertise tagline, from real data></h3>
<p align="center"><location> · <company/context></p>

<p align="center">
  <a href="<website>"><img src="https://img.shields.io/badge/Website-<domain>-e11d48?style=flat-square" alt="Website" /></a>
  <a href="<linkedin-url>"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn" /></a>
  <a href="https://github.com/<username>"><img src="https://img.shields.io/github/followers/<username>?label=Follow&style=flat-square&color=181717&logo=github&logoColor=white" alt="GitHub followers" /></a>
</p>

<p align="center">
  <one or two sentence bio, from real data, no filler><br/>
</p>
```

Notes:
- `width="100%"` on the image (not a fixed px width) — makes it fill the
  container on both desktop and mobile.
- Every badge above is a static shields.io badge — no live API call, so it
  can't go down.

---

## 6. Step 3 — Tech stack icons

Use [skillicons.dev](https://skillicons.dev), not shields.io badges, for a
denser/more visual row:

```html
<h3 align="center">Tech Stack</h3>
<p align="center">
  <img src="https://skillicons.dev/icons?i=<comma,separated,slugs>" alt="Tech stack" />
</p>
```

**Only use slugs it actually supports.** Verified by probing response size
(an unsupported slug silently returns a tiny 256-byte placeholder icon
instead of erroring — check for that, don't assume any slug works):

```bash
for icon in <candidate slugs>; do
  echo "$icon: $(curl -s "https://skillicons.dev/icons?i=$icon" | wc -c) bytes"
done
# 256 bytes = unsupported/placeholder, discard that slug
```

Confirmed **supported** in this build: `ts js nodejs html css bash git
github vscode figma vercel notion linux`

Confirmed **NOT supported** (don't use — they silently render blank):
`openai cursor gemini n8n ollama claude copilot`

If the real stack includes an unsupported tool (e.g. Claude Code, n8n,
OpenAI/Gemini APIs), mention it in the bio prose instead of as an icon.

---

## 7. Step 4 — GitHub streak stats

`github-readme-stats.vercel.app`'s main stats/top-langs endpoints are the
most commonly-linked profile widget on the internet — and consequently
**frequently return `503`** because the free shared instance is overloaded.
**Check it live before using it**:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://github-readme-stats.vercel.app/api?username=<u>"
```

If it 503s (likely), use the streak-stats variant instead — different
service, historically more reliable:

```html
<h3 align="center">GitHub Activity</h3>
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github-readme-streak-stats.herokuapp.com/?user=<username>&theme=dark&hide_border=true" />
    <img src="https://github-readme-streak-stats.herokuapp.com/?user=<username>&hide_border=true" alt="GitHub streak stats" />
  </picture>
</p>
```

---

## 8. Step 5 — Self-hosted GitHub stats visualization (jstrieb/github-stats)

This is the heaviest widget: it needs its **own separate repo**, not just a
README edit, because it needs a personal access token (a credential), which
an agent should never create or handle — only the human can do this part.

### 8.1 Setup (human does this)

1. Open [github.com/jstrieb/github-stats](https://github.com/jstrieb/github-stats)
   → **Use this template** → create `<username>/github-stats`, public.
2. [github.com/settings/tokens/new](https://github.com/settings/tokens/new)
   → classic token → scope: `repo` only → generate → copy it.
3. In the new repo: **Settings → Secrets and variables → Actions → New
   repository secret** → name `ACCESS_TOKEN` → paste the token.
4. **Actions** tab → the workflow is named **"Generate Stats Images"** (not
   "generate-images" — that's not its real name, a common wrong guess) →
   **Run workflow**. It builds from source (Zig), takes ~1–2 min.
5. It commits `overview.svg` and `languages.svg` to a branch called
   **`generated`** (not a folder on `master` — easy wrong assumption).

Verify:

```bash
curl -s "https://api.github.com/repos/<username>/github-stats/contents/?ref=generated" \
  | grep -o '"name": *"[^"]*"'
# expect overview.svg and languages.svg listed
```

### 8.2 Embedding

The generated SVGs self-adapt to GitHub's light/dark toggle via the
`#gh-dark-mode-only` / `#gh-light-mode-only` URL-fragment convention (GitHub's
own documented trick — hides/shows duplicate `<img>` tags based on site
theme). Use the **`blob`** URL form, not `raw.githubusercontent.com`, and
list all four `<img>` tags together in one block so the two cards sit side
by side instead of stacking:

```html
<p align="center">
  <img src="https://github.com/<username>/github-stats/blob/generated/overview.svg#gh-dark-mode-only" alt="GitHub stats overview" />
  <img src="https://github.com/<username>/github-stats/blob/generated/languages.svg#gh-dark-mode-only" alt="Languages used" />
  <img src="https://github.com/<username>/github-stats/blob/generated/overview.svg#gh-light-mode-only" alt="GitHub stats overview" />
  <img src="https://github.com/<username>/github-stats/blob/generated/languages.svg#gh-light-mode-only" alt="Languages used" />
</p>
```

### 8.3 Known bug #1 (we introduced it) — missing `viewBox` clips text on mobile

The generated SVGs ship as `<svg width="360" height="210">` with **no
`viewBox`**. Without one, mobile Safari doesn't scale the internal
`foreignObject`/HTML table content correctly, and long values (e.g. "Lines
of code changed: 971,516") get clipped to "971,49" on iPhone. Desktop is
unaffected.

**Fix**: patch the *generator's own workflow* (in `github-stats/.github/
workflows/main.yml`) to inject a `viewBox` into the generated files every
run, right after the "Generate images" step and before "Commit to the repo":

```yaml
    - name: Add viewBox for correct mobile scaling
      run: |
        for f in overview.svg languages.svg; do
          if [ -f "$f" ]; then
            sed -i -E 's@(<svg id="gh-[a-z-]+-mode-only"[^>]*width="([0-9]+)"[^>]*height="([0-9]+)")@\1 viewBox="0 0 \2 \3"@' "$f"
          fi
        done
```

**Critical gotcha inside the gotcha**: the file also contains small inline
icon `<svg class="octicon" ...>` elements (star, fork, etc.) that **already
have their own `viewBox`**. A naive regex like
`s@(<svg[^>]*width="([0-9]+)"[^>]*height="([0-9]+)")@\1 viewBox="0 0 \2 \3"@`
(no `id=` scoping) matches those too and injects a **second** `viewBox`
attribute onto each icon — duplicate attributes make the SVG invalid XML,
so the browser refuses to render the *entire* card (shows as a broken-image
icon). The regex above is scoped to `id="gh-[a-z-]+-mode-only"` specifically
so it only ever touches the one root tag. Verify after any change:

```bash
grep -o 'viewBox="[^"]*"' overview.svg | sort | uniq -c
grep -c '<svg' overview.svg
# the two counts should match 1:1 (one viewBox per svg tag) — if
# viewBox count > svg count, something got duplicated
```

### 8.4 Known bug #2 (upstream, unfixed) — `overview.svg` still clips on iPhone Safari

Even with a correct `viewBox`, `overview.svg` specifically has a **confirmed,
open, unresolved upstream bug**:
[jstrieb/github-stats#110](https://github.com/jstrieb/github-stats/issues/110)
— stat rows overflow on iPhone Safari, desktop is fine, `languages.svg` is
unaffected. This is a `foreignObject`+Safari layout limitation in their
generator, not something fixable by patching attributes. Decision made in
this build: **accept it** (cosmetic, one row, one browser) rather than drop
the card. Revisit if/when upstream fixes it.

### 8.5 Caching traps while debugging this

- `raw.githubusercontent.com/<owner>/<repo>/<branch>/<file>` caches per
  branch ref for several minutes. If a fix looks like it didn't take,
  fetch the exact commit SHA instead before concluding it's broken:
  `raw.githubusercontent.com/<owner>/<repo>/<commit-sha>/<file>`
- GitHub's own page rendering can also lag a few minutes after a push.
  Don't revert a fix because it "still looks broken" seconds after pushing
  — verify at the SHA-pinned raw URL first, *then* decide.

---

## 9. Step 6 — Snake contribution graph (Platane/snk)

Runs entirely inside the profile repo itself — no separate repo needed.

`.github/workflows/snake.yml`:

```yaml
name: Generate snake contribution graph

on:
  schedule:
    - cron: "0 */12 * * *"
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  generate:
    permissions:
      contents: write
    runs-on: ubuntu-latest
    steps:
      - uses: Platane/snk@v3
        id: snake-gif
        with:
          github_user_name: ${{ github.repository_owner }}
          outputs: |
            dist/github-snake.svg
            dist/github-snake-dark.svg?palette=github-dark

      - uses: crazy-max/ghaction-github-pages@v4
        with:
          target_branch: output
          build_dir: dist
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This publishes to an **`output`** branch (created automatically on first
run). Embed with light/dark variants:

```html
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/<username>/<username>/output/github-snake-dark.svg" />
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/<username>/<username>/output/github-snake.svg" />
    <img alt="GitHub contribution snake" src="https://raw.githubusercontent.com/<username>/<username>/output/github-snake.svg" />
  </picture>
</p>
```

**One-time repo setting required**: `Settings → Actions → General →
Workflow permissions → Read and write permissions → Save` (otherwise the
push at the end of the job is rejected). Do this before the first run.

This workflow just worked on the first correctly-configured run — no bugs
hit here, unlike the next one.

---

## 10. Step 7 — 3D contribution graph (yoshi389111/github-profile-3d-contrib)

`.github/workflows/profile-3d-contrib.yml`:

```yaml
name: Generate 3D contribution graph

on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: yoshi389111/github-profile-3d-contrib@latest
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          USERNAME: ${{ github.repository_owner }}
      - name: Commit & push
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add -f ./profile-3d-contrib/*.svg
          git commit -m "Generate 3D contribution graph" || exit 0
          git push
```

**Two mistakes made and fixed here — don't repeat them:**

1. **Don't pin an old version tag** (e.g. `@0.7.1`). That release predates a
   Node runtime bump; GitHub-hosted runners now instantly fail actions still
   declaring the old deprecated Node version (`node16`). Use `@latest` (or a
   current tag like `@0.9.3`, checked against the repo's actual tags at
   build time — old cached knowledge of "the latest tag" goes stale).
2. **The action takes `USERNAME`/`GITHUB_TOKEN` as `env:` vars, not `with:`
   inputs.** Its `action.yml` genuinely has no `inputs:` section at all —
   passing `with: username: ...` is silently ignored, and the mis-pinned
   version above was actually what caused the visible failure, masking this
   second issue until the version was fixed too. Always check the target
   action's own README/`action.yml` for its real interface instead of
   assuming a `with:` pattern from a similar action.

It commits generated SVGs directly onto `main` (not a separate branch), into
`profile-3d-contrib/`. Available theme filenames (confirmed present after a
real run — check yours, names can change between action versions):
`profile-gitblock.svg`, `profile-green-animate.svg`, `profile-green.svg`,
`profile-night-green.svg`, `profile-night-rainbow.svg`,
`profile-night-view.svg`, `profile-season-animate.svg`, `profile-season.svg`,
`profile-south-season-animate.svg`, `profile-south-season.svg`. There is
**no** `profile-day-compact.svg` in current versions — don't assume a
filename, list the actual directory contents first:

```bash
curl -s "https://api.github.com/repos/<username>/<username>/contents/profile-3d-contrib" \
  | grep -o '"name": *"[^"]*"'
```

Embed (pick a light-friendly file for light mode, a dark one for dark mode):

```html
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/<username>/<username>/main/profile-3d-contrib/profile-night-rainbow.svg" />
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/<username>/<username>/main/profile-3d-contrib/profile-green.svg" />
    <img alt="3D contribution graph" src="https://raw.githubusercontent.com/<username>/<username>/main/profile-3d-contrib/profile-night-rainbow.svg" />
  </picture>
</p>
```

---

## 11. Step 8 — Quote block: static text, not a widget

`quotes-github-readme.vercel.app` (a common "awesome README" pick) was
tried and dropped. Its `horizontal` variant ships as
`<svg width="600" height="auto">` — **`height="auto"` is not a valid SVG
length**, so different browsers compute a wrong/inconsistent intrinsic
height for the `<img>`, causing text to clip on mobile and the card to
render smaller than full-width on desktop. Its `vertical` variant is
correctly formed (`300×300` fixed) but can't be full-width by design either.

Rather than depend on a third fragile foreignObject-SVG generator, use a
plain static quote — same visual effect, zero failure modes, and genuinely
full-width since it's just text:

```html
<p align="center">
  <em>"<quote text>"</em><br/>
  <sub>— <attribution></sub>
</p>
```

(Verify the quote and attribution are real/correctly attributed before
using — this build reused a real, correctly-attributed Fred Brooks quote
that had already been rendered by the widget once, rather than inventing
one.)

---

## 12. Step 9 — Footer

```html
<p align="center">
  <sub><a href="<website>">website</a> · <a href="https://github.com/<username>">GitHub</a> · <a href="<linkedin>">LinkedIn</a> · <a href="mailto:<email>">Email</a></sub>
</p>
```

---

## 13. Git workflow gotcha — read this before your second push

`snake.yml` and `profile-3d-contrib.yml` both trigger on `push: branches:
main` **and** commit their generated output back onto `main` (or `output`)
at the end of the run. That means: every time you push a README edit, both
workflows re-run and push a bot commit back — so your **next** push will be
rejected with `! [rejected] main -> main (fetch first)` unless you rebase
first. This happened repeatedly during this build. Standard fix, every time:

```bash
git add <files>
git commit -m "<message>"
git fetch origin
git rebase origin/main
git push
```

If you'd rather not deal with this every time, drop the `push:` trigger
from both workflows (keep `schedule:` + `workflow_dispatch:`) — trades
"images update instantly when you push" for "no more rebase dance." Not
done in this build (kept the instant-update behavior on purpose), but worth
considering.

---

## 14. Quick-start checklist (condensed)

1. Confirm repo is `<username>/<username>`, image committed and pushed
2. Pull real bio data: GitHub API + personal site (§4)
3. Write skeleton: image (100% width) → tagline → badges → bio (§5)
4. Add tech stack via skillicons.dev, verified slugs only (§6)
5. Add streak-stats card, not the often-503 main stats endpoint (§7)
6. (Optional, needs human for the token) Set up `github-stats` template repo,
   embed with `#gh-dark/light-mode-only`, patch its workflow for `viewBox`
   scoped to the root `<svg id="gh-...">` tag only (§8)
7. Add `snake.yml`, enable Actions read/write permission, embed from
   `output` branch (§9)
8. Add `profile-3d-contrib.yml` with `@latest` + `env:` inputs, embed from
   `main` branch, verify real filenames first (§10)
9. Add a static quote, not a live quote widget (§11)
10. Add footer links (§12)
11. Every push after step 6+: `git fetch && git rebase origin/main && git push` (§13)
12. `curl` every image URL before declaring done

---

## 15. Reference: this build's final `README.md`

The exact file this repo shipped with as of 2026-09-20, for diffing against
future changes or reuse as a template (replace all `<username>`/personal
details before reusing for someone else):

```html
<div align="center">
  <img src="Khalidhasananik.jpg" width="100%" alt="Khalid Hasan" />
</div>

<h3 align="center">Project Coordination · Web Development · AI Agents · Workflow Automation</h3>
<p align="center">Dhaka, Bangladesh · Building at AWTOMATIG</p>

<p align="center">
  <a href="https://khalidhasananik.com"><img src="https://img.shields.io/badge/Website-khalidhasananik.com-e11d48?style=flat-square" alt="Website" /></a>
  <a href="https://www.linkedin.com/in/khalidhasananik/"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn" /></a>
  <a href="https://github.com/khalidhasananik"><img src="https://img.shields.io/github/followers/khalidhasananik?label=Follow&style=flat-square&color=181717&logo=github&logoColor=white" alt="GitHub followers" /></a>
</p>

<p align="center">
  I coordinate projects and build AI agent &amp; automation workflows at AWTOMATIG —<br/>
  pairing tools like Claude Code, n8n, and the OpenAI/Gemini APIs with full-stack web development to ship things end-to-end.
</p>

<br/>

<h3 align="center">Tech Stack</h3>
<p align="center">
  <img src="https://skillicons.dev/icons?i=ts,js,nodejs,html,css,bash,git,github,vscode,figma,vercel,notion" alt="Tech stack" />
</p>

<br/>

<h3 align="center">GitHub Activity</h3>
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github-readme-streak-stats.herokuapp.com/?user=khalidhasananik&theme=dark&hide_border=true" />
    <img src="https://github-readme-streak-stats.herokuapp.com/?user=khalidhasananik&hide_border=true" alt="GitHub streak stats" />
  </picture>
</p>

<p align="center">
  <img src="https://github.com/khalidhasananik/github-stats/blob/generated/overview.svg#gh-dark-mode-only" alt="GitHub stats overview" />
  <img src="https://github.com/khalidhasananik/github-stats/blob/generated/languages.svg#gh-dark-mode-only" alt="Languages used" />
  <img src="https://github.com/khalidhasananik/github-stats/blob/generated/overview.svg#gh-light-mode-only" alt="GitHub stats overview" />
  <img src="https://github.com/khalidhasananik/github-stats/blob/generated/languages.svg#gh-light-mode-only" alt="Languages used" />
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/output/github-snake-dark.svg" />
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/output/github-snake.svg" />
    <img alt="GitHub contribution snake" src="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/output/github-snake.svg" />
  </picture>
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/main/profile-3d-contrib/profile-night-rainbow.svg" />
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/main/profile-3d-contrib/profile-green.svg" />
    <img alt="3D contribution graph" src="https://raw.githubusercontent.com/khalidhasananik/khalidhasananik/main/profile-3d-contrib/profile-night-rainbow.svg" />
  </picture>
</p>

<p align="center">
  <em>"When a task cannot be partitioned because of sequential constraints, the application of more effort has no effect on the schedule. The bearing of a child takes nine months, no matter how many women are assigned."</em><br/>
  <sub>— Fred Brooks</sub>
</p>

<p align="center">
  <sub><a href="https://khalidhasananik.com">khalidhasananik.com</a> · <a href="https://github.com/khalidhasananik">GitHub</a> · <a href="https://www.linkedin.com/in/khalidhasananik/">LinkedIn</a> · <a href="mailto:khalidhasananik@outlook.com">Email</a></sub>
</p>
```

---

## 16. Open items / accepted limitations

- `overview.svg` clips one row on iPhone Safari specifically — upstream bug,
  tracked at
  [jstrieb/github-stats#110](https://github.com/jstrieb/github-stats/issues/110),
  accepted as-is (§8.4).
- `snake.yml` / `profile-3d-contrib.yml` still trigger on every push to
  `main`, requiring a rebase before the next push (§13) — left this way on
  purpose to keep images fresh instantly; revisit if the rebase dance gets
  annoying.
