---
name: creator-site-github-pages
category: Web and publishing
description: Build and deploy a personal creator/portfolio site (tabs like Home, Portfolio, Blog, Video, Contact) that live-fetches the person's blog and YouTube content, hosted free on GitHub Pages with their own custom domain, accessible and light/dark aware, with an optional skills showcase. Use when someone wants a personal site, link-hub, or creator homepage tied to their own blog/YouTube/socials.
---

# Creator site on GitHub Pages

A playbook for building a single-page personal/creator site — tabs, not a long scroll — that pulls real content from the person's own blog and YouTube channel automatically, and deploying it free on GitHub Pages under their own domain. Built from an end-to-end session doing exactly this; the notes below are the things that weren't obvious the first time.

## 1. Gather requirements up front

Ask for (or infer from context) before building:
- Sections wanted (common set: Home, Portfolio, Blog, Video, Contact) and what each should link to
- Brand name / site title
- Real links: blog URL, YouTube channel URL/handle, LinkedIn, GitHub, other socials
- Domain name and registrar (affects the DNS instructions given later)
- Any photo for the Home/About area
- Design preference if stated (color, tone); otherwise make deliberate choices per a frontend-design skill if one is available — avoid generic AI-design tells (cream+terracotta, Inter everywhere, numbered-tab navs presented as a design flourish rather than real navigation, etc.)

Don't fabricate portfolio projects, testimonials, or video titles. Pull real post titles/links from the actual blog (fetch it) before writing the Blog section, and leave anything you can't verify (like a LinkedIn URL not given) as a clearly-marked placeholder rather than guessing a real-looking one.

## 2. Build as a single HTML file with real tabs, not anchor-scroll

People expect clicking a nav item to show only that section — not jump to a spot on one long page. Implement actual client-side tabs:
- Each section is `<section id="home" class="frame">`, all but the first marked `hidden`
- Nav links are `<a href="#home">`, styled as real buttons (background + border), not bare text — a plain numbered-tab look (`00 home`, `01 portfolio`) reads as decorative/templated; style them as clickable pills/buttons instead
- A single delegated click handler intercepts any in-page `a[href^="#"]` (nav buttons *and* any in-content CTA like a hero "Read the blog" button), updates `location.hash` via `history.pushState`, and toggles `hidden` on sections
- Handle `popstate` for back/forward, and read `location.hash` on load so the page can be deep-linked to a tab

This is a real deliverable file (`index.html`), not something built inside a sandboxed in-chat preview — because it needs `fetch()` to external feeds and (optionally) YouTube `<iframe>` embeds, both of which a sandboxed preview environment typically blocks. Build and iterate on it as a plain file.

## 3. Live content: fetch patterns and their gotchas

**Blog posts — easy, do this directly in the page's own JS:**
Blogger exposes a public, CORS-friendly JSONP feed, no key needed:
```
https://<blog>.blogspot.com/feeds/posts/default?alt=json-in-script&max-results=4&callback=myCallback
```
Load it with a dynamically-created `<script>` tag; `myCallback(feed)` receives `feed.feed.entry[]`, each with `.title.$t`, `.link[]` (filter `rel === 'alternate'` for the post URL), `.published.$t`, and `.summary.$t`/`.content.$t` for an excerpt (strip HTML tags, truncate, escape before inserting).

**YouTube videos — do NOT fetch this directly from the browser.** Two things go wrong if you do:

1. **A public CORS proxy (e.g. `allorigins.win`) will eventually fail for real visitors.** Ad-blockers, browser privacy modes, and corporate networks commonly block known proxy/relay domains. It may work when you test it and then silently break for the actual user — don't rely on client-side proxy-fetching YouTube's RSS feed as the production solution.
2. **The channel RSS feed (`https://www.youtube.com/feeds/videos.xml?channel_id=UC...`) is capped at the ~15 most recent uploads, mixed Shorts and long-form.** If the person posts a lot of Shorts, filtering those out of only 15 items can leave far fewer long-form videos than wanted.

**The reliable fix: do the fetching server-side via a scheduled GitHub Action, commit the result as a small JSON file in the repo, and have the page fetch that file same-origin.** This sidesteps CORS entirely (nothing to block) and isn't capped at 15 recent items.

- To get *only long-form videos* (no Shorts), don't use the RSS feed at all — query the channel's own **Videos tab**, which YouTube itself already excludes Shorts from (Shorts live under a separate Shorts tab):
  ```yaml
  - name: Set up yt-dlp
    run: pip install --quiet --no-input yt-dlp
  - name: Fetch latest long-form videos
    run: |
      yt-dlp --flat-playlist --playlist-end 5 --dump-single-json \
        "https://www.youtube.com/@<handle>/videos" > raw.json
      python3 - <<'PY'
      import json
      data = json.load(open("raw.json"))
      videos = [{"id": e["id"], "title": e["title"]} for e in data.get("entries", [])[:5] if e.get("id") and e.get("title")]
      json.dump(videos, open("videos.json", "w"), indent=2)
      PY
      rm -f raw.json
  ```
- If Shorts inclusion is actually wanted, the RSS feed is simpler and fine.
- A full workflow: checkout → set up yt-dlp → run the fetch script above → commit `videos.json` if changed (`git diff --cached --quiet || git commit ...`) → push. Give it `permissions: contents: write`, trigger on a `schedule` (e.g. every 6 hours) and `workflow_dispatch` for manual runs.
- After first creating the workflow, trigger it immediately (`gh api repos/<owner>/<repo>/actions/workflows/<file>.yml/dispatches -X POST -f ref=main`), poll `gh api repos/<owner>/<repo>/actions/runs --jq '.workflow_runs[0]'` until `status` is `completed`, then `git pull` to fetch the generated JSON into the local clone — don't make the person wait for the first scheduled run.
- Page-side: `fetch('videos.json', { cache: 'no-store' }).then(r => r.json())`, render thumbnails from `https://i.ytimg.com/vi/<id>/hqdefault.jpg`, and swap a clicked thumbnail for a `<iframe src="https://www.youtube.com/embed/<id>?autoplay=1">` (click-to-play, not auto-embedding every video up front).

**General principle:** any "show my latest X automatically" feature for a real deployed site should prefer a same-origin file over a client-side fetch to a third party that might rate-limit, require a key, or get blocked. A scheduled Action + committed JSON is the pattern; reach for it by default, not as a later fix.

## 4. Deploying to GitHub Pages + a custom domain

1. The repo must be named exactly `<github-username>.github.io` for root-domain Pages hosting. If the person doesn't have it yet, they create it (empty, public) themselves and grant access to it — repo creation and account-wide access are often outside what an assistant session can do on its own, even when it can push to a repo once attached.
2. Clone with `git clone --depth 1 https://github.com/<owner>/<repo>.git <path>` — give it a generous timeout (large packs through a proxied connection can take minutes).
3. Copy in the site files, commit, push.
4. Add a `CNAME` file at the repo root containing just the bare domain (e.g. `example.com`) — this is what tells GitHub Pages to serve the custom domain, and GitHub auto-provisions free HTTPS for it once DNS resolves. A repo named `<user>.github.io` usually has Pages auto-enabled; otherwise it's Settings → Pages → Deploy from branch → `main` / root.
5. Give the person these DNS records to add at their registrar:
   - Four **A** records on the apex/root (`@`) pointing at: `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - One **CNAME** record on `www` pointing at `<user>.github.io`
6. After DNS is added, verify with a DNS-over-HTTPS lookup (e.g. `https://dns.google/resolve?name=<domain>&type=A`) rather than guessing — confirms propagation without waiting blindly.
7. Expect an SSL hostname-mismatch error on `https://` for a while right after DNS first resolves — that's GitHub still issuing the certificate (can take hours), not a misconfiguration. `http://` or fetching the `<user>.github.io` URL (which redirects to the custom domain once the CNAME file is live) are both good ways to confirm the setup worked before HTTPS finishes provisioning.

## 5. Quality bar (check before calling it done)

Reviewing a finished site against Apple's Human Interface Guidelines turned up the same handful of issues. Build these in from the start instead of fixing them later:

- **Tap targets ≥ 44 px tall** on nav links, buttons and filter chips (`min-height:44px` plus horizontal padding). Chips that aren't links shouldn't look clickable.
- **Small text ≥ 14 px.** Chips, footer, dates, eyebrow labels and mono nav text drift down to 11–12 px; keep body ~17 px and everything else 14 px or more.
- **Light and dark both.** Define colours as CSS variables on `:root` (including the translucent nav background and the text-on-accent colour) and override them under `@media (prefers-color-scheme: light)`. Darken the accent for light mode and check it: white on the accent should be ≥ 4.5:1 (an amber like `#e2a63b` needs to become roughly `#9a5f00`). Don't add a manual toggle.
- **Accent restraint.** Spend the accent colour on the primary button and one or two brand touches. Mark the active nav tab with a neutral fill and an accent underline rather than a solid accent pill.
- **Phone nav.** On narrow widths make the nav one horizontally scrolling row (`flex-wrap:nowrap; overflow-x:auto`, hide the scrollbar) instead of wrapping onto two lines.
- **Labels.** Icon-only links get `aria-label`. An image inside a labelled button (for example a video thumbnail in a `role="button"` with an `aria-label`) correctly has an empty `alt`; don't flag or "fix" those. Keep visible `:focus-visible` styles and a `prefers-reduced-motion` guard.
- **Verify, don't assume.** Preview locally (`python3 -m http.server`), emulate a 375 px viewport in both colour schemes, and check computed sizes and `documentElement.scrollWidth === innerWidth` rather than judging from a screenshot.

## 6. Optional: a Skills tab that builds itself from a repo

If the person publishes Claude skills in a public repo (`skills/<name>/SKILL.md`), the site can list them with no build step by reading the repo from the browser: `GET https://api.github.com/repos/<owner>/<repo>/contents/skills` for the folders, then each `SKILL.md` from `raw.githubusercontent.com` and parse its frontmatter.

- **Sections.** Add a one-line `category: <Section name>` to each `SKILL.md` frontmatter. Group cards by it under a mono eyebrow heading, in a fixed preferred order with unknown categories alphabetical and "Other" last. Add filter chips (All plus one per section) as real `<button aria-pressed>` elements that toggle the `hidden` attribute on sections; give `.section[hidden]{display:none}` its own rule so it wins over `display:grid`.
- **Preview images.** Put a 1280×720 `preview.svg` (or PNG) in each skill folder and show it 16:9 at the top of the card from `raw.githubusercontent.com/<owner>/<repo>/main/skills/<name>/preview.svg`. SVG needs no image tooling: generate it from a small script in the site's colours with a simple illustration of what the skill does, its name and a one-line tagline. Use system font stacks (Georgia, Menlo) because web fonts don't load inside an `<img>` SVG. If the image 404s, remove its `src` so the card still shows a neutral tile.
- **Descriptions** in frontmatter are long trigger text; clamp them to ~4 lines with `-webkit-line-clamp`.
- **Caveat:** unauthenticated GitHub API calls are limited to 60 per hour per visitor IP, so a shared office network could see the "couldn't load" fallback. Keep the fallback link to the repo, and if it matters, generate a `skills.json` with a scheduled Action (same pattern as `videos.json`) and fetch that same-origin instead.

## 7. Pushing from an assistant session

HTTPS pushes fail with `Password authentication is not supported`, and a personal access token pasted into a terminal prompt is easy to paste in the wrong place (it ends up in shell history and chat). Prefer the browser device flow:

1. With the person's OK, `brew install gh` (a download, so ask first).
2. `gh auth login --web --hostname github.com --git-protocol https --skip-ssh-key`: the person opens `https://github.com/login/device` and enters the one-time code it prints. No token ever passes through the assistant.
3. `gh auth setup-git`, then plain `git push` works. Verify the result with `git push` output (`<old>..<new> main -> main`) and a cache-busted `curl` of the live page for a marker string.
4. If a token was exposed anyway, tell the person to delete it at `github.com/settings/tokens`.

## 8. Iterating after launch

Treat this as a living repo, not a one-off file: small requests ("only show 5 videos", "remove Shorts", "swap the photo") are each a short edit-commit-push-redeploy cycle, with the GitHub Action re-triggered manually (`workflow_dispatch`) when the fetch logic itself changes so the fix is visible immediately rather than on the next schedule.
