# Content Engine Pipeline — Project Documentation

> **Last updated:** June 21, 2026
> **Authors:** Manny (execution) × Masa (strategy, content, research)
> **Status:** Planning complete, POC implementation starting

---

## 1. Mission

A **learn-and-earn content engine**: deep-dive trending AI/agent/deep-tech topics, then turn each deep-dive into **podcasts and videos** that build an audience and generate income toward financial freedom.

The deep-dive teaches Manny the frontier; the published asset monetizes that knowledge. Both must happen per topic.

---

## 2. Architecture Overview

```
GitHub Actions (orchestrator — public repo, free unlimited minutes)
  ├── Pi Harness (AI reasoning within stages — judgment calls)
  ├── E2B / Daytona / FreeStyle (CPU-heavy sandbox work — scraping, Agent-Reach)
  ├── Modal.com (GPU work — video rendering, lip-sync, Flux thumbnails)
  └── GitHub Actions runner (lightweight stuff — file ops, ffmpeg mixing)
```

### Two-repo structure
| Repo | Visibility | Purpose |
|------|-----------|---------|
| `manigandanp/content-engine-pipeline` | **Public** | GitHub Actions workflows, README, non-secret config. Free unlimited runner minutes. |
| `manigandanp/content-engine-skills` | **Private** | All stage skills (Python scripts). Checked out by the public repo's workflows using a PAT. |

### Pipeline stages (10 stages)
```
01-news-aggregation     → Horizon: fetch, score, enrich
02-topic-selection      → Pi: pick stories, group into segments
03-script-writing       → Pi: write Manny×Masa dialogue
04-tts-generation       → Mistral Voxtral: per-segment TTS
05-audio-mastering      → ffmpeg: mix + loudnorm to -16 LUFS
06-slack-delivery       → Post to Slack for approval
07-video-assembly       → Modal: avatars, lip-sync, visuals → MP4
08-thumbnail            → Flux on Modal: cover art
09-publish              → YouTube, podcast RSS, X/Twitter, blog
10-approval-gate        → Slack buttons → trigger publish
```

---

## 3. Key Decisions

### D1: GitHub Actions as orchestrator (not compute engine)
- **Decision:** GitHub Actions schedules and orchestrates the pipeline. Heavy compute (GPU video rendering, TTS) is delegated to external services (Modal, Mistral API) called from within workflows.
- **Rationale:** Public repos get free unlimited GitHub-hosted runner minutes. But runners are x86_64 with 6-hour job limits — not suitable for GPU work. Use them for orchestration; delegate heavy lifting.
- **Risk:** GitHub ToS says Actions should be for "software production, testing, deployment, or publication." Content generation is arguably not that. Mitigated by having a self-hosted runner fallback plan (Epic K3).

### D2: Pi Harness for AI reasoning, not orchestration
- **Decision:** Pi (earendil-works/pi) handles AI judgment within stages (topic selection, script writing, visual planning). GitHub Actions remains the top-level orchestrator.
- **Rationale:** Using both Pi and GitHub Actions as orchestrators creates a two-layer orchestration problem. Pi's value is tool-calling + LLM reasoning for editorial decisions, not scheduling.
- **Integration:** Pi runs as a step in the GitHub Actions workflow — `pi run --skill topic-selection --input briefing.md`.

### D3: Skills as Python scripts in a private repo (not Hermes SKILL.md)
- **Decision:** Each pipeline stage is a self-contained Python script in the private repo `content-engine-skills/`. The public repo's workflows check out the private repo using a PAT.
- **Rationale:** Protects IP (scripts, prompts, logic). Public repo only contains the workflow YAML and non-secret config. Requires a GitHub PAT with `repo` scope stored as `PRIVATE_REPO_PAT` secret.

### D4: Horizon for news aggregation, Mistral API for scoring
- **Decision:** Use Horizon (Thysrael/Horizon) for fetching, scoring, and enriching news. Use `mistral-small-latest` via Mistral API direct (`api.mistral.ai`) for AI scoring.
- **Rationale:** Horizon is proven working (180 commits, active maintenance, MCP integration). A self-hosted LLM proxy was tested but blocked by Cloudflare from Docker containers (OpenAI SDK's httpx TLS fingerprint) and virtual keys expired after ~10 minutes. Mistral direct API has no such issues.
- **Reddit:** JSON API is 403-blocked from data center IPs. Workaround: add subreddits as RSS feeds (`https://www.reddit.com/r/{sub}/hot.rss`). r/LocalLLaMA and r/artificial still 429 rate-limited even via RSS — needs fixing in Epic G1.

### D5: Agent-Reach for X/Twitter and Reddit access (selective)
- **Decision:** Use Agent-Reach (Panniantong/agent-reach) for accessing X/Twitter (cookie-based twitter-cli) and Reddit (rdt-cli with cookies). Install in CI, not on VPS.
- **Rationale:** Horizon needs paid Apify ($5/month) for Twitter. Agent-Reach does it free with cookies. Cookies last months — stored as base64-encoded GitHub Secrets, refreshed monthly. Not a daily problem.
- **Scope:** Only install the channels we need (twitter, reddit, youtube transcripts). Don't install the full suite (Bilibili, XiaoHongShu, V2EX are irrelevant).

### D6: Pre-segmented audio for lip-sync
- **Decision:** Use the per-segment MP3 files (`segments/seg_000.mp3` through `seg_NNN.mp3`) for lip-syncing, not the full 11-minute audio.
- **Rationale:** Most open-source lip-sync tools (SadTalker, MuseTalk) handle 5-15 second clips well. Each segment is a single speaker turn, typically 5-15 seconds. This solves the duration problem completely. We know which speaker from `seg_durations.txt`.

### D7: Selective host appearance (not 100% talking head)
- **Decision:** Show host avatars only when necessary (cold open, story intros, key takeaways, outro). Show sources, text highlights, code snippets, and screen recordings for the rest.
- **Rationale:** Less GPU usage (40% lip-sync vs 100%), more engaging (visual rhythm), more credible (showing actual sources). Overlays generated with PIL/ffmpeg (CPU, no GPU).

### D8: Slack-based approval (not GitHub Environments)
- **Decision:** POC uses GitHub Environments for the approval gate (simplest to wire). Real pipeline upgrades to Slack interactive buttons — "Approve" / "Reject" in the Slack message itself.
- **Rationale:** Manny lives in Slack. Better UX than going to GitHub to click "Review deployments." Slack buttons trigger `repository_dispatch` event to start the publish workflow.

### D9: Podcast distribution via RSS (not per-platform upload)
- **Decision:** Host podcast audio once (Castopod self-hosted or Anchor), generate an RSS feed, submit that feed to Apple Podcasts / Spotify / Amazon Music / etc.
- **Rationale:** Upload once, distribute everywhere. No need to manually upload to each platform. RSS is the podcast industry standard.

### D10: Docker-first convention
- **Decision:** All third-party tools/services run in Docker containers, not bare-metal on the VPS. Security isolation — Manny's standing rule.
- **Applied to:** Horizon (Docker on VPS). Will apply to Castopod and any future self-hosted services.
- **Note:** In GitHub Actions, Docker is optional (runner is already isolated). Horizon runs directly with `uv sync && uv run horizon` in CI.

### D11: Free-tier resource layering
- **Decision:** Use free tiers across multiple platforms to keep monthly cost at $0.
| Resource | Platform | Free tier |
|----------|----------|-----------|
| Pipeline orchestration | GitHub Actions (public repo) | Unlimited minutes |
| LLM (scoring + script) | Mistral API | Free tier |
| TTS | Mistral Voxtral / Edge TTS | Free |
| GPU (video, lip-sync, thumbnails) | Modal.com | 2 accounts × $30/month = $60/month |
| CPU sandbox (scraping, Agent-Reach) | E2B / Daytona / FreeStyle | Free tiers |
| Podcast hosting | Castopod (self-hosted) | Free |
| Code hosting | GitHub | Free |

---

## 4. Phased Delivery

### Part 1 — POC (Sprint 1, 1 week)
**Goal:** Prove the full pipeline loop works end-to-end with the cheapest possible version of each stage.

**What POC includes:**
- Horizon news aggregation in CI
- Simple score-threshold topic selection
- Single-prompt Mistral script generation
- Voxtral TTS + ffmpeg mastering
- Slack delivery for approval
- GitHub Environment approval gate
- GitHub Release as publish stub

**What POC deliberately skips:**
- Lip-sync, avatars, video
- Pi harness (uses simple logic instead)
- Multi-platform publishing
- Agent-Reach (uses Horizon's built-in sources only)
- Enhanced aggregation (Reddit fix, Twitter, YouTube transcripts)

**POC Definition of Done:** You wake up, see a podcast in Slack, click approve in GitHub, and it lands as a downloadable GitHub Release — no terminal touched.

### Part 2 — Real Pipeline (Sprints 2-5, 6-10 weeks)

| Sprint | Epics | Duration | Deliverable |
|--------|-------|----------|-------------|
| Sprint 2 | G + H1-H2 + J1 | 1-2 weeks | Enhanced aggregation + simple visuals + YouTube public upload |
| Sprint 3 | F + H3-H4 | 2-3 weeks | Pi harness + avatar generation + lip-sync R&D |
| Sprint 4 | H5-H6 + I | 1-2 weeks | Screen recordings + video assembly + thumbnails |
| Sprint 5 | J2-J3 + K | 1-2 weeks | Multi-platform publish + robustness + Slack buttons |

**Critical insight:** Sprint 2 delivers a *publishing* content engine (audio + simple video). Everything in Sprint 3+ is *quality enhancement*. You can start building an audience after Sprint 2 and improve production quality in parallel.

---

## 5. Proven Assets (don't reinvent)

| Asset | Location | Status |
|-------|----------|--------|
| Horizon Docker setup | `~/projects/Horizon/` on VPS | ✅ Working, daily cron at 5:30 AM + 4:30 PM IST |
| Podcast TTS + mastering scripts | `~/.hermes/skills/creative/podcast-production/scripts/` | ✅ Proven (38/38 segments, 0 failures) |
| Test podcast episode | `~/data/2026-06-21-news-roundup-test/podcast_final.mp3` | ✅ 11.4 min, -16.71 LUFS, Apple Podcasts compliant |
| Background music | `~/data/2026-06-10-world-news-podcast/bg_music.mp3` | ✅ Royalty-free, reusable |
| Hermes cron jobs | Morning (6 AM) + Evening (5 PM) IST | ✅ Running, delivering to Slack |
| Slack delivery | Slack channels (IDs in GitHub Secrets) | ✅ Proven working |
| Modal models | LongCat (avatar), Wan (video), Flux (images) | ✅ Configured, MODAL_TOKEN in ~/.hermes/.env |
| Mistral API | `MISTRAL_API_KEY` in ~/.hermes/.env | ✅ Working for both scoring and TTS |
| Voice IDs | MANNY + MASA (Voxtral) | ✅ Locked, see MEMORY.md |

---

## 6. Credentials & Secrets

> **All secrets are stored as GitHub Secrets or in `~/.hermes/.env` on the VPS.** No secret values are committed to any repo. Secret names are listed here for reference only.

### VPS (`~/.hermes/.env`)
All API keys, tokens, and channel IDs are stored in this file on the VPS. Refer to the file directly for the full list.

### GitHub Secrets (to be set in `content-engine-pipeline` repo — Task A4)
The following secrets need to be stored in the repo settings for the CI pipeline:
- `PRIVATE_REPO_PAT` — GitHub PAT for cloning the private skills repo
- `MISTRAL_API_KEY` — AI scoring + TTS
- `SLACK_BOT_TOKEN` — Slack delivery
- `SLACK_CHANNEL_IDS` — Slack channel IDs (comma-separated)
- `GITHUB_TOKEN` — For GitHub events source (Epic G2)
- `MODAL_TOKEN_ID` / `MODAL_TOKEN_SECRET` — Video rendering (Epic H)
- `YOUTUBE_CLIENT_ID` / `YOUTUBE_CLIENT_SECRET` / `YOUTUBE_REFRESH_TOKEN` — YouTube upload (Epic J1)

---

## 7. Known Issues & Pitfalls

1. **Self-hosted LLM proxy:** Cloudflare blocks OpenAI SDK's httpx TLS fingerprint from Docker containers. Virtual keys expire after ~10 minutes. **Use Mistral API direct instead.**
2. **Reddit JSON API:** 403-blocked from data center IPs. r/LocalLLaMA and r/artificial 429 even via RSS. **Fix in Epic G1.**
3. **GitHub events:** Return 0 items without auth (60 req/hr unauthenticated limit). **Fix in Epic G2.**
4. **write_file tool redacts secrets:** When writing .env files via the Hermes write_file tool, secret values are replaced with `***`. Must copy keys via terminal/Python, not write_file.
5. **Voxtral API returns JSON with base64 audio_data:** Not raw MP3 bytes. Must decode base64 before writing to file.
6. **Lip-sync for 11+ min audio:** Most open-source tools handle short clips. Solved by using pre-segmented audio (Decision D6).
7. **GitHub ToS risk:** Actions should be for "software production." Content generation is borderline. Fallback: self-hosted runner ($0.002/min as of March 2026).

---

## 8. Project Board

- **Board:** https://github.com/users/manigandanp/projects/4
- **Public repo:** https://github.com/manigandanp/content-engine-pipeline
- **Private repo:** https://github.com/manigandanp/content-engine-skills
- **52 issues:** 11 epics, 12 POC stories, 18 real pipeline stories
- **Labels:** `POC`, `real-pipeline`, `epic`, `infrastructure`, `news-aggregation`, `script-writing`, `audio-production`, `video-production`, `publishing`, `approval`, `robustness`
