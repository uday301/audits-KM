# Claude Deployment Setup Guide

## Overview
This file contains all the details Claude needs to push HTML pages live to Hostinger automatically via GitHub.

---

## GitHub Repository
- **Repo URL:** https://github.com/uday301/audits-KM.git
- **Branch:** main
- **Owner:** uday301
- **Repo Name:** audits-KM

## GitHub Access Token
- **Token:** [STORED PRIVATELY — paste your GitHub PAT when starting a new chat]
- **Scopes:** repo (full access)
- **Where to find it:** GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)

## Hostinger Site
- **Live Domain:** https://audits.kerkarmedia.com
- **Old Temp URL:** https://slategray-wren-882514.hostingersite.com (still works but use custom domain)
- **Deploy Path:** public_html (root)
- **Auto Deployment:** Active via GitHub webhook

## How It Works
1. Claude pushes an HTML file to the GitHub repo
2. GitHub webhook triggers Hostinger auto-deployment
3. File goes live instantly at its own URL

---

## URL Structure
| File Name | Live URL |
|---|---|
| index.html | https://audits.kerkarmedia.com |
| any-file.html | https://audits.kerkarmedia.com/any-file.html |

---

## Instructions for Claude (Paste this in a new chat)
Hey Claude! I have a website deployment setup. Here are the details:

- **GitHub Repo:** https://github.com/uday301/audits-KM.git
- **GitHub Token:** [PASTE YOUR TOKEN HERE]
- **Live Site:** https://audits.kerkarmedia.com
- **How to deploy:** Clone the repo using the token, add the HTML file with a unique filename, commit and push. Hostinger auto-deploys via webhook.

Please push [FILE NAME] as [NEW FILE NAME].html to the repo and give me the live URL.

---

## Pages Published So Far
| File Name | Live URL | Date |
|---|---|---|
| index.html | https://audits.kerkarmedia.com | 2026-04-23 |
| chaibymira-digital-marketing-roadmap.html | https://audits.kerkarmedia.com/chaibymira-digital-marketing-roadmap.html | 2026-04-23 |
| fpa-digital-marketing-roadmap.html | https://audits.kerkarmedia.com/fpa-digital-marketing-roadmap.html | 2026-04-23 |
| km-dashboard.html | https://audits.kerkarmedia.com/km-dashboard.html | 2026-05-08 |

---

## Branding — Kerkar Media
- **Primary font:** DM Sans (headings) + Inter (body)
- **Colors:** #003DCC (CTA blue), #035398 (brand blue), #124648 (footer green), #E9E9E8 (light gray)
- **Button radius:** 10px
- **Moodboard:** See Kerkar_Media_Moodboard.pdf in project files
- **Animations:** GSAP + ScrollTrigger for all page animations
- **Charts:** Chart.js v4

## Dashboard Naming Convention
Client dashboards follow this pattern:
- File: `[client-slug]-dashboard.html`
- URL: `https://audits.kerkarmedia.com/[client-slug]-dashboard.html`

Example:
- `km-dashboard.html` → Kerkar Media internal
- `chaibymira-dashboard.html` → Chai by Mira client

---

## Git Commands Claude Uses
```bash
git clone https://[TOKEN]@github.com/uday301/audits-KM.git
git config user.email "claude@anthropic.com"
git config user.name "Claude"
git add .
git commit -m "Add [page name]"
git push https://[TOKEN]@github.com/uday301/audits-KM.git main
```
# Last deployed: Fri May  8 13:10:47 UTC 2026
