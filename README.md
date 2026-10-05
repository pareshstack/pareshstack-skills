# Paresh's Stack — Skills

Reusable AI assistant skills from [Paresh's Stack](https://pareshstack.com) — playbooks built while doing the actual work, cleaned up and packaged so anyone can install and use them.

A **skill** here is a folder containing a `SKILL.md`: a focused, battle-tested set of instructions for one kind of task, written for an AI coding/chat assistant (such as Claude) to follow. Point your assistant at one and it knows the steps, the gotchas, and the fixes — without you having to re-explain them or rediscover the same mistakes.

- 📺 YouTube: [@pareshstack](https://www.youtube.com/@pareshstack)
- ✍️ Blog: [pareshstack.blogspot.com](https://pareshstack.blogspot.com)
- 💼 LinkedIn: [in/itspareshsharma](https://www.linkedin.com/in/itspareshsharma/)

---

## Skills in this repo

| Skill | What it does |
|---|---|
| [`creator-site-github-pages`](skills/creator-site-github-pages) | Build and deploy a personal creator/portfolio site (Home, Portfolio, Blog, Video, Contact tabs) that live-fetches your blog and YouTube content, hosted free on GitHub Pages with your own domain. |
| [`shorts-maker`](skills/shorts-maker) | Turn a long tutorial/talking video into several clean vertical YouTube Shorts (no burned-in captions) plus a matching glossy thumbnail for each clip. |

More skills get added here over time as they're built and proven out. Check back, or watch the repo.

## Installing a skill

Each skill lives in its own folder under [`skills/`](skills) and is just a `SKILL.md` file (plus sometimes a short `README.md` with usage notes). How you install it depends on your setup:

**Claude Code / Claude apps that support custom skills:**
Copy the skill's folder into your assistant's skills directory, for example:
```bash
# project-level (this project only)
cp -r skills/creator-site-github-pages /path/to/your/project/.claude/skills/

# user-level (all your projects)
cp -r skills/creator-site-github-pages ~/.claude/skills/
```
Reload or start a new session and the skill is available — your assistant will use it automatically when a request matches its description, or you can invoke it by name.

**If your tooling has a skill-installer CLI** that can pull a skill straight from a GitHub repo, point it at this repo and the skill folder you want (check your specific tool's docs for the exact command — this varies by platform).

**Just want to read it?** Every `SKILL.md` is plain Markdown — open it directly and follow the steps yourself, no installation required.

## Using a skill once installed

You generally don't need to do anything special — just describe what you want in plain language, and a well-installed skill gets picked up automatically based on its description. For example, with `creator-site-github-pages` installed:

> "I want a personal site with tabs for my work, blog, and videos, deployed on GitHub Pages with my own domain."

## What's generic here (and what isn't)

Every skill in this repo is written to be reused by anyone — no personal data, accounts, domains, or credentials of mine are baked in. Placeholders like `<handle>`, `<owner>/<repo>`, and `<your-domain>` mark the spots where your own details go. If you ever see something that looks like it should have been a placeholder and isn't, please open an issue.

## Contributing

This repo is maintained by Paresh as a personal project. Suggestions, fixes, and skill ideas are welcome via issues or pull requests — if you adapt one of these for a different platform (e.g. a non-Blogger blog, a non-GitHub host) and want to share it back, open a PR.

## License

[MIT](LICENSE) — use these however you like, no attribution required (though it's always appreciated).
