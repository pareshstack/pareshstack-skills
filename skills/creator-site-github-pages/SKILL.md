---
name: creator-site-github-pages
description: Build and deploy a personal creator/portfolio site (tabs like Home, Portfolio, Blog, Video, Contact) that live-fetches the person's blog and YouTube content, hosted free on GitHub Pages with their own custom domain. Use when someone wants a personal site, link-hub, or creator homepage tied to their own blog/YouTube/socials.
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

## 5. Iterating after launch

Treat this as a living repo, not a one-off file: small requests ("only show 5 videos", "remove Shorts", "swap the photo") are each a short edit-commit-push-redeploy cycle, with the GitHub Action re-triggered manually (`workflow_dispatch`) when the fetch logic itself changes so the fix is visible immediately rather than on the next schedule.
