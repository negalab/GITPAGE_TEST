# HANDOFF — Rebuild MR_5PM's Wiki + Two-Site Pages from scratch

> **For an AI agent in a fresh session.** This document contains everything needed to rebuild this exact system without re-asking the user. Read top-to-bottom before starting. The user has already made every routine decision — they are documented here. Only stop to confirm with the user when explicitly told to (auth, public push).

---

## 0. What you're building

A **personal wiki + dual GitHub Pages publishing system** for the user **MR_5PM**.

```
Single source of truth:  /WIKI/*.md  (Obsidian vault)
                              │
                              ▼
            scripts/sync-wiki.mjs  (frontmatter-aware fan-out)
                  │                          │
                  ▼                          ▼
   sites/public/  (Astro Fuwari)    sites/personal/  (Quartz v4)
   ── publish: true only            ── all WIKI files
   ── editorial Helmut Newton       ── obsidian wiki-link/graph
      design (cream paper, dark
      ink, Cormorant + Pinyon
      Script + Noto Serif KR)
                  │                          │
                  └──────── single ──────────┘
                  GitHub Actions workflow
                  combines into one Pages site:
                  / (public) and /notes/ (personal)
```

Live URLs after deploy:
- **Public**: https://negalab.github.io/GITPAGE_TEST/
- **Personal**: https://negalab.github.io/GITPAGE_TEST/notes/

---

## 1. User profile — established facts (don't re-ask)

| Item | Value |
|------|-------|
| Display name | MR_5PM |
| Tone | formal Korean (~습니다), starts with "자," 70%+ |
| Primary work | AI content creation, app dev, lectures |
| GitHub username | **negalab** (NOT dngkwon-prog — that's a separate gh CLI account) |
| GitHub email | dykwon@smit.ac.kr |
| Local path | `/Users/letsdooyoung/GITPAGE_TEST` |
| Obsidian Vault | This same folder is the vault |
| Voice input | Often uses voice-to-text — apply MR_5PM voice coach footer |

**Behavioral memories** (read these too — at `~/.claude/projects/-Users-letsdooyoung-GITPAGE-TEST/memory/`):
- Foundation-first iteration is fine — ugly intermediate states OK
- Subscriber-facing visual work needs multiple polished options, NOT a single default
- This project's full state is logged in `project_gitpage_test.md`

---

## 2. Decisions already locked in (don't re-ask)

| # | Question | User's answer |
|---|----------|---------------|
| 1 | All file format | `.md` (Obsidian managed) |
| 2 | Categorization | Auto-classify by content (CLAUDE.md has rules) |
| 3 | INDEX layout | Both — latest-first table + per-category sections |
| 4 | RAW originals after move | Move to `RAW/_archived/` (never delete) |
| 5 | Both sites | Public on GitHub Pages |
| 6 | Public-page selection | frontmatter `publish: true` flag |
| 7 | Font | Pretendard (personal site) + editorial Cormorant stack (public site) |
| 8 | Newsletter | Buttondown placeholder (user activates separately) |
| 9 | URL structure | Single repo, public at `/`, personal at `/notes/` |
| 10 | SSG public | Astro Fuwari → then re-skinned with editorial Helmut Newton design |
| 11 | SSG personal | Quartz v4 |
| 12 | Korean font in editorial | Noto Serif KR fallback (paired with Cormorant) |
| 13 | welcome.md language | Korean |
| 14 | Masthead content | MR_5PM brand, not the Helmut Newton placeholder text |

---

## 3. Prerequisites — verify before starting

```bash
node --version       # need 20+
npm --version        # need 9+
git --version        # any
gh --version         # 2.50+ recommended
```

**`gh` CLI auth** — the user has TWO accounts. `negalab` MUST be active before `gh repo create` / `gh api` calls.

```bash
gh auth status
# if negalab not present:
gh auth login         # interactive — user does this in browser
# if present but not active:
gh auth switch --user negalab
```

If `gh auth login --user negalab` fails with `unknown flag: --user` — that flag was removed; just run `gh auth login` (no flag) and follow the prompts. The user knows this.

---

## 4. Build phases

Tackle in order. Each phase has verification.

### Phase 1 — Foundation files (10 min, no external calls)

Create the directory structure and the 3 governance docs.

```bash
mkdir -p RAW/_archived WIKI sites scripts .github/workflows
touch RAW/.gitkeep RAW/_archived/.gitkeep WIKI/.gitkeep
```

Write these files (full contents below in §10 File reference, or copy from the existing repo if you have access):
- `CLAUDE.md` — agent operating rules: trigger words ("정리해", "공개해", etc.), category rules, frontmatter format with `publish: false` default, INDEX update protocol, what NOT to do
- `README.md` — user-facing usage docs
- `INDEX.md` — initial empty index with table headers + category sections
- `WIKI/welcome.md` — sample post with `publish: true` (MR_5PM intro in Korean)
- `WIKI/wiki-system-design.md` — sample post with `publish: false` (system meta)

Frontmatter schema for all WIKI files:
```yaml
---
title: "..."
category: "AI|Tech|Business|Education|Content|Personal|Reference|Idea"
tags: [t1, t2, t3]
raw_source: "RAW/_archived/..."   # empty string if not from RAW
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
summary: "..."
publish: false                     # CRITICAL: false default; user must opt in
cover: ""
---
```

**Verify**: `ls -la` shows the 3 dirs + 3 docs + 2 sample WIKI files.

---

### Phase 2 — Clone the two SSGs (5 min)

```bash
cd sites
git clone --depth 1 https://github.com/saicaca/fuwari.git public
git clone --depth 1 https://github.com/jackyzha0/quartz.git personal
rm -rf public/.git public/.github personal/.git personal/.github
cd ..
```

**Customize Fuwari** (`sites/public/src/config.ts`):
- `siteConfig.title`: "MR_5PM"
- `siteConfig.subtitle`: "AI · 콘텐츠 · 개발에 대한 기록"
- `siteConfig.lang`: "ko"
- `siteConfig.themeColor`: `{ hue: 30, fixed: true }`
- `profileConfig.name`: "MR_5PM"
- `profileConfig.bio`: Korean bio
- `profileConfig.avatar`: `"assets/images/mr5pm-monogram.svg"` (you'll create this in Phase 6)
- `profileConfig.links`: GitHub link to https://github.com/negalab
- `navBarConfig.links`: Home, Archive, About, then external link `{ name: "📓 Notes", url: "/GITPAGE_TEST/notes/", external: true }`

**Customize Fuwari** (`sites/public/astro.config.mjs`):
- `site`: `"https://negalab.github.io"`
- `base`: `"/GITPAGE_TEST/"`

**Clear Fuwari demo posts**:
```bash
rm -rf sites/public/src/content/posts/*
touch sites/public/src/content/posts/.gitkeep
```

**Customize Quartz** (`sites/personal/quartz.config.ts`):
- `pageTitle`: "MR_5PM Notes"
- `analytics`: `null`
- `locale`: "ko-KR"
- `baseUrl`: `"negalab.github.io/GITPAGE_TEST/notes"`
- `theme.typography`: `{ header: "Inter", body: "Inter", code: "JetBrains Mono" }` — **DO NOT use "Gowun Dodum" — Google Fonts lacks weight 700, breaks build**

**Customize Quartz** (`sites/personal/quartz/styles/custom.scss`):
```scss
@use "./base.scss";

// Pretendard font is loaded via <link> in Head.tsx — NOT @import here.
// @import url(...) breaks the build because it must precede all rules,
// but base.scss above already emits CSS rules.

:root {
  --bodyFont: "Pretendard Variable", Pretendard, -apple-system, BlinkMacSystemFont, system-ui, "Apple SD Gothic Neo", "Noto Sans KR", sans-serif;
  --headerFont: "Pretendard Variable", Pretendard, -apple-system, system-ui, sans-serif;
}
body { font-family: var(--bodyFont); }
h1, h2, h3, h4, h5, h6 { font-family: var(--headerFont); font-weight: 700; letter-spacing: -0.02em; }
article p, article li { line-height: 1.75; }
```

**Inject Pretendard into Quartz** (`sites/personal/quartz/components/Head.tsx`):
Find the `<link rel="preconnect" href="https://cdnjs.cloudflare.com" ... />` line and insert AFTER it:
```jsx
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css"
/>
```

**Verify**: `ls sites/public/src/config.ts sites/personal/quartz.config.ts` exists with edits.

---

### Phase 3 — Sync script (10 min)

Write `scripts/sync-wiki.mjs`. Behavior:
1. Read all `WIKI/*.md`
2. For each, parse frontmatter
3. Write to `sites/personal/content/{file}.md` always (Quartz reads our frontmatter natively)
4. If `publish === true`, also write to `sites/public/src/content/posts/{file}.md` with frontmatter rewritten to Fuwari's schema
5. Generate `sites/personal/content/index.md` listing all entries (Quartz needs a homepage)

**Critical detail #1 — date emission**:
Fuwari's content collection schema parses `published` as **Date type**. YAML strings like `"2026-05-04"` (quoted) FAIL validation. Date-shaped values must be emitted **bare**:
```yaml
published: 2026-05-04        # ✅ correct
published: "2026-05-04"      # ❌ Astro fails: "Expected type 'date', received 'string'"
```
In serializeFrontmatter, detect `/^\d{4}-\d{2}-\d{2}$/` and emit unquoted.

**Critical detail #2 — Fuwari frontmatter mapping**:
| WIKI field | Fuwari field |
|-----------|--------------|
| `title` | `title` |
| `created` | `published` (bare date) |
| `summary` | `description` |
| `tags` | `tags` |
| `category` | `category` |
| `publish: true` | `draft: false` |
| `cover` (if non-empty) | `image` |

**Critical detail #3 — `content/index.md` generation**:
Quartz fails to build without `content/index.md`. The sync clears the dir each run, so a static index.md gets wiped. Therefore **the sync script must write index.md after copying**, listing all entries with publish-status badges (🔓 / 🔒) using Obsidian wiki-link `[[slug|title]]` syntax.

Reference implementation: see existing `scripts/sync-wiki.mjs` in this repo. ~120 LOC, no external deps (minimal regex-based YAML parser).

**Verify**:
```bash
node scripts/sync-wiki.mjs
# expect:
# ✓ Synced N files → sites/personal/content/
# ✓ Generated content/index.md with N entries
# ✓ Synced M files → sites/public/src/content/posts/ (publish: true only)
```

---

### Phase 4 — GitHub Actions workflow (5 min)

Write `.github/workflows/deploy.yml`. Single job that:

1. Checkout
2. Setup Node 22
3. Setup pnpm 9 (Fuwari uses pnpm; lockfile is `pnpm-lock.yaml`)
4. Run `node scripts/sync-wiki.mjs`
5. `cd sites/public && pnpm install --frozen-lockfile && pnpm build`
6. `cd sites/personal && npm ci && npx quartz build` (Quartz uses npm; lockfile `package-lock.json`)
7. Combine: `cp -r sites/public/dist/* _site/` then `mkdir _site/notes && cp -r sites/personal/public/* _site/notes/`
8. `actions/upload-pages-artifact@v3` → `actions/deploy-pages@v4`

Reference: existing `.github/workflows/deploy.yml`. Permissions: `contents: read`, `pages: write`, `id-token: write`. Concurrency group `pages`.

Also write `.gitignore`:
```
.DS_Store
node_modules/
sites/public/dist/
sites/public/.astro/
sites/personal/public/
sites/personal/.quartz-cache/
*.log
.env*
```

---

### Phase 5 — Init git, create repo, push (USER CONFIRMATION REQUIRED)

```bash
git init -b main
git config user.email "dykwon@smit.ac.kr"
git config user.name "negalab"
git add -A
git commit -m "Initial commit — RAW→WIKI pipeline + two-site Pages publishing"
```

**STOP HERE — confirm with user before pushing.** This makes the work public on the internet. State explicitly what will be public (the welcome.md, the system files) and ask "push해도 될까요?"

After user says yes:
```bash
gh repo create negalab/GITPAGE_TEST --public --source=. --remote=origin --push --description "Personal wiki + two-site GitHub Pages publishing"
gh api -X POST /repos/negalab/GITPAGE_TEST/pages -f 'build_type=workflow'
```

The `gh api` call enables Pages with GitHub Actions as source.

**Watch the first build** — first builds usually fail because of the issues in §6. Use:
```bash
gh run watch --repo negalab/GITPAGE_TEST
```
or poll:
```bash
until [ "$(gh run list --repo negalab/GITPAGE_TEST --limit 1 --json status --jq '.[0].status')" = "completed" ]; do sleep 15; done
```

---

### Phase 6 — Apply Helmut Newton editorial design

The default Fuwari design is a generic dark-mode-friendly blog template. The user explicitly rejected it ("디자인 엄청 구리다"). Replace the visual layer with the Helmut Newton editorial design from claude.ai/design (bundle ID `ZF6FbadLGLUU5sSudCqNmw`).

**Strategy**: Don't restructure Fuwari's components. Layer a CSS override on top + add a masthead component for the home page only. This is much safer and faster than rewriting Fuwari.

Steps:

#### 6a. Add fonts to Fuwari head

Edit `sites/public/src/layouts/Layout.astro`. Inside `<head>`, after the `<meta name="generator">` line, add:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,500;0,600;0,700;1,400;1,500;1,600&family=Cormorant+SC:wght@500;600;700&family=Libre+Caslon+Text:ital,wght@0,400;0,700;1,400&family=Pinyon+Script&family=Noto+Serif+KR:wght@400;500;600;700&display=swap" rel="stylesheet" />
```

#### 6b. Replace top of `sites/public/src/styles/main.css`

Replace the existing `@import url(...pretendard...)` and `:root { --font-sans... }` block with the editorial design system block (`--paper`, `--ink`, `--font-serif`, `--font-serif-sc`, `--font-script`, `--font-label`, tracking/leading vars). Set `html, body { font-family: var(--font-serif); background: var(--paper-deep); color: var(--ink); }`.

#### 6c. Append editorial overrides to bottom of `main.css`

A ~270 LOC override section that:
- Sets `html, body` background to `--paper-deep`
- Flattens `.card-base` (no rounded corners, no shadow, paper bg, top border)
- Restyles `.card-base a[class*="text-3xl"]` titles → Cormorant SC uppercase
- Restyles card descriptions → italic Cormorant
- Hijacks `--primary` → `var(--ink)` so all Fuwari accent color references map to ink
- Restyles `#navbar` → top/bottom 1px rules, label font, uppercase
- Applies `filter: grayscale(1) contrast(1.05)` to all `.card-base img`, `#post-cover img`, `.custom-md img`
- Restyles `.custom-md` body: first paragraph as lede (centered italic), h1 as huge SC, h2 as section label with bottom rule, blockquote as pull quote (large SC uppercase), hr as 33%-width centered rule
- Drops rounded corners from buttons, panels, pagination
- Footer in label font

Reference: existing `sites/public/src/styles/main.css` lines 187+. Copy verbatim.

#### 6d. Create `sites/public/src/components/EditorialMasthead.astro`

Magazine masthead for home page only. Three-column grid (badge-left | title-stack | badge-right), title stack has "MR" (huge SC) / "five pm" (Pinyon Script) / "_5PM" (huge SC). Side badges in italic Cormorant. Below: lede paragraph with profileConfig.bio. Bottom: thin rule. Mobile: collapse to single column.

Reference: existing `sites/public/src/components/EditorialMasthead.astro`.

#### 6e. Inject masthead on home page only

Edit `sites/public/src/pages/[...page].astro`:
```jsx
import EditorialMasthead from "../components/EditorialMasthead.astro";
// ...
<MainGridLayout>
  {page.url.prev === undefined && <EditorialMasthead />}
  <div class="editorial-section-label-wrap"><span class="editorial-label">LATEST DISPATCHES</span></div>
  <PostPage page={page}></PostPage>
  <Pagination ... />
  {page.url.prev === undefined && <Newsletter />}
</MainGridLayout>
```
The `page.url.prev === undefined` check ensures the masthead only shows on page 1, not paginated archive pages.

#### 6f. Rewrite `sites/public/src/components/Newsletter.astro`

Replace with editorial-style version: 3-column grid (badge | body | badge), thick top rule, Cormorant SC headline with Pinyon Script accent, lede paragraph, ink-on-paper button. Buttondown form with placeholder username. Reference: existing file.

---

### Phase 7 — Replace placeholder avatar (5 min)

Fuwari's `demo-avatar.png` is an anime-style cartoon that clashes with the editorial design. Replace with a self-contained SVG monogram.

Write `sites/public/src/assets/images/mr5pm-monogram.svg`:
- 400×400 viewBox
- Cream paper background + dot pattern + subtle grain filter
- Top + bottom horizontal rules (thin and thick)
- Top label "VOL. I · ISSUE 001" in italic serif
- Center: huge "MR" in serif (`Georgia` family — generic serif so it renders as `<img>`; webfonts don't apply to SVG loaded as image)
- Below: "five pm" in cursive (`'Brush Script MT', 'Snell Roundhand', cursive` family)
- Mid divider rule
- Bottom: "a journal of A.I. · content · code"
- Footer: "SPRING · 2026"

**Critical**: SVGs loaded via `<img src="*.svg">` render in their own context with NO access to web fonts loaded by parent HTML. Use only generic font families (`serif`, `cursive`) or system fonts (`Georgia`, `'Brush Script MT'`).

Update `sites/public/src/config.ts`:
```ts
profileConfig.avatar = "assets/images/mr5pm-monogram.svg"
```

Append to `main.css` editorial overrides:
```css
#sidebar .card-base a[href*="/about/"] {
    border-radius: 0 !important;
    border: 1px solid var(--rule);
}
#sidebar .card-base a[href*="/about/"] img {
    filter: none !important;     /* SVG is already monochrome */
    border-radius: 0 !important;
}
#sidebar .card-base .font-bold {
    font-family: var(--font-serif-sc) !important;
    text-transform: uppercase;
    letter-spacing: var(--headline-tracking);
}
#sidebar .card-base .h-1 { display: none !important; }
```

---

## 5. Final commit + push

```bash
node scripts/sync-wiki.mjs        # ensure synced state
git add -A
git commit -m "..."
git push
```

Watch the build. The third+ deploy should succeed (assuming you fixed the build failures preemptively — see §6).

---

## 6. Build failures to expect (and how we fixed them)

The first three builds in the original session all failed. **Avoid them by writing correct code from the start**, but here's the reference if they recur:

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | Astro: `posts → welcome data does not match collection schema. published: Expected type "date", received "string"` | Sync emitted `published: "2026-05-04"` (quoted) | In serializeFrontmatter, detect `/^\d{4}-\d{2}-\d{2}$/` and emit bare (no quotes) |
| 2 | Quartz: `Failed to fetch font Gowun Dodum with weight 700, got Bad Request` | Google Fonts has Gowun Dodum at weight 400 only | Switch typography to Inter (or any font with all weights) |
| 3 | Quartz: `Failed to emit from plugin ComponentResources: @import rules must precede all rules aside from @charset and @layer statements` | Put `@import url(...pretendard...)` AFTER `@use "./base.scss"` in custom.scss | Move font loading to `<link>` in Head.tsx instead; SCSS can't have `@import url()` after other rules |
| 4 | Quartz: `Warning: you seem to be missing an index.md home page file at the root of your content folder` | Sync wiped content/ each run, no homepage | Have sync script generate content/index.md after copying entries |

---

## 7. Verification checklist

After final push and successful build:

```bash
# Both URLs return 200
curl -sI https://negalab.github.io/GITPAGE_TEST/      | head -1
curl -sI https://negalab.github.io/GITPAGE_TEST/notes/ | head -1

# Public site has editorial markers
curl -s https://negalab.github.io/GITPAGE_TEST/ | grep -oE '(Cormorant|Pinyon|MR_5PM)' | sort -u

# Personal site has Quartz markers
curl -s https://negalab.github.io/GITPAGE_TEST/notes/ | grep -oE '(MR_5PM Notes|Pretendard)' | sort -u
```

Visual checks (open both URLs in browser):
- ✅ Public site: cream paper background, MR / five pm / _5PM masthead at top, "LATEST DISPATCHES" label, post cards as flat magazine entries with serif uppercase titles, Newsletter card at bottom, Cormorant fonts everywhere
- ✅ Public site: profile avatar in sidebar shows the monogram SVG (NOT the anime cartoon)
- ✅ Personal site: Quartz default layout, Pretendard font for Korean, content index page with 🔓/🔒 badges
- ✅ Both: images appear in grayscale on public, normal on personal

---

## 8. Things the user explicitly DOES NOT want

- ❌ Default Fuwari design (rejected as "구리다")
- ❌ The drag-to-position blocks from the design source bundle (it's a prototype-only feature)
- ❌ The Tweaks panel from the design source bundle
- ❌ localStorage layout persistence
- ❌ Any third-party analytics enabled by default
- ❌ Auto-pushing without explicit confirmation (public action)
- ❌ Auto-flipping `publish: false` → `true` without user saying "공개해 [filename]"
- ❌ Deleting RAW originals (must move to `_archived/`)

---

## 9. Things to ASK the user (only these)

1. **Before `gh repo create` + first push**: "이대로 인터넷에 공개됩니다. push할까요?" — wait for explicit yes
2. **If gh CLI auth is missing or wrong account active**: walk them through `gh auth login` (the `--user` flag does NOT exist on `gh auth login`, only on `gh auth switch`)
3. **If a WIKI file already exists with the same target name during sync**: ask "병합 / 새 파일 / 스킵?"
4. **If a category is unclear during RAW→WIKI organize**: present 2 candidates

Everything else: assume the answer from §2.

---

## 10. File reference (paths, sizes, purpose)

| Path | Purpose | LOC ≈ |
|------|---------|-------|
| `CLAUDE.md` | Agent rules: triggers, frontmatter spec, INDEX update protocol | 200 |
| `README.md` | User-facing usage docs | 130 |
| `INDEX.md` | Two-section content index, manually + sync-updated | 50 |
| `HANDOFF.md` | This file | — |
| `RAW/` | User inbox (.gitkeep) | — |
| `RAW/_archived/` | Originals after sync | — |
| `WIKI/welcome.md` | Sample post, publish: true, Korean intro | 30 |
| `WIKI/wiki-system-design.md` | Sample post, publish: false, system meta | 60 |
| `scripts/sync-wiki.mjs` | WIKI → both sites, publish flag filter | 130 |
| `sites/public/` | Astro Fuwari (cloned + heavily customized) | — |
| `sites/public/src/config.ts` | Site title, profile, navbar | 80 |
| `sites/public/astro.config.mjs` | site URL + base path | 170 |
| `sites/public/src/styles/main.css` | Base + editorial overrides | 480 |
| `sites/public/src/layouts/Layout.astro` | Adds editorial fonts to head | 570 |
| `sites/public/src/components/EditorialMasthead.astro` | Home page magazine masthead | 145 |
| `sites/public/src/components/Newsletter.astro` | Editorial-style Buttondown card | 175 |
| `sites/public/src/pages/[...page].astro` | Home page; injects masthead + label + newsletter | 30 |
| `sites/public/src/assets/images/mr5pm-monogram.svg` | Avatar replacement | 50 |
| `sites/personal/` | Quartz v4 (cloned + customized) | — |
| `sites/personal/quartz.config.ts` | Site title, baseUrl, fonts (Inter), locale | 100 |
| `sites/personal/quartz/styles/custom.scss` | Pretendard typography overrides | 25 |
| `sites/personal/quartz/components/Head.tsx` | Pretendard `<link>` injection | 110 |
| `.github/workflows/deploy.yml` | Single workflow: sync + build both + combine + deploy Pages | 65 |
| `.gitignore` | Excludes node_modules, builds, .DS_Store, .env | 30 |

---

## 11. Operational rhythm (after rebuild — for the user's day-to-day)

1. **Add content**: drop `.md` file into `RAW/`
2. **Trigger organize**: tell Claude "정리해" — Claude reads CLAUDE.md rules, parses RAW files, writes structured `WIKI/{slug}.md` with `publish: false`, moves originals to `RAW/_archived/`, updates INDEX.md
3. **Selectively publish**: tell Claude "공개해 {slug}" or directly edit `publish: true` — only flagged files appear on the public site
4. **Deploy**: `node scripts/sync-wiki.mjs && git add -A && git commit -m "..." && git push` — GitHub Actions builds both sites in 5–7 min

---

## 12. If you (the AI) get stuck

- The user accepts ugly intermediate states — ship a working pipeline first, polish later
- For visual choices, present multiple options with previews; never install one default and stop
- Auth/push/destructive actions: always confirm
- Build failures: check §6 first; the same 4 issues recur

When in doubt, read the user's memory entries at `~/.claude/projects/-Users-letsdooyoung-GITPAGE-TEST/memory/MEMORY.md`. Those are the durable behavioral rules established across sessions.

---

**End of handoff.** A fresh AI following this document should produce a functionally identical system in ~2 hours of work, with one user pause for `gh auth` (if needed) and one for push confirmation.
