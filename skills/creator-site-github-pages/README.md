# creator-site-github-pages

Build and deploy a personal creator/portfolio site — Home, Portfolio, Blog, Video, Contact as real tabs — that live-fetches your blog posts and YouTube videos automatically, hosted free on GitHub Pages under your own domain.

## What it does

- Builds a single-page site with proper tab navigation (no endless scrolling)
- Live-fetches your latest blog posts (Blogger feed, no API key)
- Live-fetches your latest long-form YouTube videos, Shorts excluded, via a scheduled GitHub Action (no API key, no fragile third-party CORS proxy)
- Applies an accessibility and design quality bar (44 px tap targets, readable text, light and dark mode)
- Optionally adds a Skills tab that groups your published Claude skills into sections with preview images, straight from a GitHub repo
- Walks through deploying it on GitHub Pages with a custom domain, including the exact DNS records you need at your registrar

## Why this exists

Built from doing this exact workflow end-to-end and hitting (then fixing) the parts that aren't obvious the first time: a public CORS proxy that looked fine in testing but got blocked for real visitors, a RSS feed that caps out before you can filter Shorts, and the GitHub Pages + custom domain + HTTPS sequence that looks broken for a few hours when it isn't.

## How to use it

Install the skill (see the repo's main [README](../../README.md) for installation), then just ask your assistant for what you want, e.g.:

> "Build me a personal site with a Home, Portfolio, Blog, and Video tab, and set it up on GitHub Pages with my domain."

The skill guides the build, the content-fetching setup, and the deployment steps — adapted to your actual blog, channel, and domain.

## Notes

- Works for any blog on Blogger. A different blogging platform needs a different feed/fetch step — ask your assistant to adapt it.
- The YouTube piece uses `yt-dlp` inside a GitHub Action, not the YouTube Data API, so there's no API key or quota to manage.
- You'll need your own GitHub account, a repo named `<your-username>.github.io`, and (optionally) a custom domain from any registrar.
