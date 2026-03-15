# 🛡️ RepoGuardian — Autonomous AI Code Review System

<div align="center">

![RepoGuardian Banner](https://img.shields.io/badge/RepoGuardian-Autonomous%20Code%20Review-3b82f6?style=for-the-badge&logo=shield&logoColor=white)

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![React](https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react&logoColor=black)](https://reactjs.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Groq](https://img.shields.io/badge/LLM-Groq%20%2B%20Llama3-f97316?style=flat-square)](https://groq.com)
[![License](https://img.shields.io/badge/License-MIT-22c55e?style=flat-square)](LICENSE)

**RepoGuardian is a multi-agent AI system that automatically reviews every Pull Request on GitHub — detecting security vulnerabilities, code smells, missing documentation, outdated dependencies, and more — then posts inline comments, raises auto-fix PRs, tracks developer patterns over time, and serves everything through a real-time dashboard.**

[Live Demo](#-dashboard) · [Quick Start](#-quick-start) · [Agents](#-agents) · [Architecture](#-architecture) · [API](#-api-reference)

</div>

---

## 📸 Dashboard Preview

> Real-time metrics, wavy health trend chart, live agent feed, per-developer memory profiles, and an AI voice assistant — all in one view.

```
┌─────────────────────────────────────────────────────────────────┐
│  🛡 RepoGuardian   Overview  Findings  Agents  History          │
│                                          ● Live · 18:52:30       │
├──────────┬──────────┬──────────┬─────────────────────────────────┤
│  30      │   1      │  84      │  313                            │
│  Avg     │  PRs     │  Critical│  Total Findings                 │
├──────────┴──────────┴──────────┴─────────────────────────────────┤
│  〜 Repo Health Trend              Recent Pull Requests          │
│  ●30 ────────────────              PR #1  X Rejected  30        │
│                                    PR #2  ⚠ Needs Work  65      │
├─────────────────────────┬───────────────────────────────────────┤
│  ◎ Selected PR Analysis │  ⚡ Live Agent Activity               │
│  [Score Ring: 30]       │  ● Security Agent  — SQL_INJECTION    │
│  Critical ████████  5   │  ● Docs Agent      — missing docstr.. │
│  High     ██████    5   │  ● Auto-Fix Agent  — Fix PR raised    │
└─────────────────────────┴───────────────────────────────────────┘
```

---

## ✨ Features

| Feature | Description |
|---|---|
| **7 Autonomous Agents** | PR Review, Security, Docs, Dependency, Complexity, Memory, Auto-Fix |
| **Inline GitHub Comments** | Every agent posts comments directly on the PR diff |
| **Auto-Fix PRs** | Raises a ready-to-merge fix PR for simple issues automatically |
| **Developer Memory** | Tracks each developer's patterns across all PRs over time |
| **Health Scoring** | Unified 0–100 score with grade (A–F) per PR |
| **Real-Time Dashboard** | React dashboard with live feed, wavy trend chart, agent status |
| **Voice Agent** | Ask questions about your team in plain English — voice or text |
| **PDF Reports** | Full per-developer performance report with charts |
| **Chat Interface** | Built-in AI chatbot powered by your memory store |

---

## 🤖 Agents

### 1. 👁 PR Review Agent
Analyses code logic, naming conventions, structure, and coding standards using **Groq Llama3-70b**. Posts inline comments on every affected line. Chunks large diffs to stay within rate limits.

### 2. 🔐 Security Agent
Two-stage scanner — first runs **20+ regex rules** (SQL injection, hardcoded secrets, eval, os.system, weak hashes, JWT none algorithm), then sends the diff to LLM for deep OWASP Top 10 analysis. Validates all line numbers against the actual PR diff before posting.

### 3. 📦 Dependency Agent
Runs **pip-audit** against `requirements.txt` in the PR. Detects known CVEs and advisory issues. No LLM needed — runs in a background thread while other agents run.

### 4. 📝 Docs Agent
Uses **Python AST** to detect public functions and classes missing docstrings. Checks README coverage and inline comment density. Combines static analysis with LLM for deeper coverage gaps.

### 5. 🧠 Complexity Agent
Uses **Radon** to calculate cyclomatic complexity per function. Flags functions above threshold with specific refactoring suggestions.

### 6. 💾 Memory Agent
Reads all findings after every PR and updates a per-developer JSON profile. Detects **streaks** (same issue 3 PRs in a row), **recurring patterns** (3+ total occurrences), and **improvements** (issue disappeared). Posts a personalised report to the PR.

### 7. 🔧 Auto-Fix Agent
Applies **8 safe deterministic fixes** and raises a ready-to-merge PR:
- `eval()` → `ast.literal_eval()`
- `DEBUG=True` → `DEBUG=False`
- `hashlib.md5` → `hashlib.sha256`
- `hashlib.sha1` → `hashlib.sha256`
- `http://` → `https://`
- `except:` → `except Exception as e:`
- Hardcoded secrets → `os.getenv()`
- `random.randint` → `secrets.randbelow()`

---

## 🏗️ Architecture

```
GitHub PR Opened
      │
      ▼
webhook/receiver.py  ←── FastAPI webhook + dashboard API
      │
      ▼
agents/orchestrator.py  ←── runs all agents, calculates unified score
      │
      ├── [Background]  tools/dependency.py   (pip-audit, no LLM)
      ├── [LLM 1/4]     agents/pr_review.py   (t=0s)
      ├── [LLM 2/4]     tools/security.py     (t=20s)
      ├── [LLM 3/4]     tools/docs.py         (t=40s)
      ├── [LLM 4/4]     tools/complexity.py   (t=60s)
      ├── [Post-score]  tools/memory.py       (no LLM)
      └── [Final]       tools/autofix.py      (no LLM → raises fix PR)
      │
      ▼
db/memory_store.json  ←── developer profiles, findings, scores
      │
      ▼
dashboard/  ←── React dashboard (Vite + Recharts)
      │
      ▼
voice_agent/  ←── reads memory_store.json, answers manager questions
```

### Health Score Formula

```
Score = 100 - Σ(severity_weight × agent_multiplier)  [min 30, max 100]

Severity weights:   high=15,  medium=7,  low=2
Agent multipliers:  security=1.5,  dependency=1.3,  pr_review=1.0,  docs=0.5
```

---

## 📁 Project Structure

```
repo_guardian/
├── agents/
│   ├── orchestrator.py        # Main coordinator — runs all agents
│   └── pr_review.py           # PR Review Agent
├── tools/
│   ├── security.py            # Security Agent (static + LLM)
│   ├── dependency.py          # Dependency Agent (pip-audit)
│   ├── docs.py                # Docs Agent (AST + LLM)
│   ├── complexity.py          # Complexity Agent (Radon)
│   ├── memory.py              # Memory Agent (pattern tracking)
│   ├── autofix.py             # Auto-Fix Agent (raises fix PRs)
│   └── report_agent.py        # PDF Report Generator
├── webhook/
│   └── receiver.py            # FastAPI server (webhook + dashboard API)
├── dashboard/                 # React dashboard (Vite)
│   └── src/
│       └── App.jsx
├── voice_agent/
│   └── voice_agent.py         # Voice/text Q&A from memory store
├── db/
│   ├── models.py              # Shared Finding / ReviewResult dataclasses
│   ├── memory_store.json      # Developer profiles (auto-generated)
│   └── last_run.json          # Last orchestrator run metadata
├── reports/                   # Generated PDF reports
├── main.py                    # Entry point
└── .env                       # API keys (never commit this)
```

---

## ⚡ Quick Start

### 1. Clone & Install

```bash
git clone https://github.com/codewithVamshi5/repo_guardian.git
cd repo_guardian
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure `.env`

```env
GITHUB_TOKEN=ghp_your_token_here
GROQ_API_KEY=gsk_your_key_here
LLM_MODEL=groq
GITHUB_REPO=codewithVamshi5/repo_guardian
```

| Variable | Where to get it |
|---|---|
| `GITHUB_TOKEN` | [GitHub Settings → Developer Settings → Personal Access Tokens](https://github.com/settings/tokens) — needs `repo` scope |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) — free tier works |

### 3. Run a PR Review

```bash
# Review PR #1 in your repo
python agents/orchestrator.py codewithVamshi5/repo_guardian 1
```

This will:
1. Run all 7 agents against PR #1
2. Post inline comments to the PR on GitHub
3. Raise a fix PR if auto-fixable issues are found
4. Write findings to `db/memory_store.json`
5. Post a memory report with developer pattern alerts

### 4. Start the Dashboard

```bash
# Terminal 1 — backend API
uvicorn webhook.receiver:app --reload --port 8000

# Terminal 2 — frontend
cd dashboard
npm install
npm run dev
```

Open **http://localhost:5173** — the dashboard loads with real data automatically.

### 5. Generate a PDF Report

```bash
# Report for all developers
python tools/report_agent.py

# Report for one developer
python tools/report_agent.py --user codewithVamshi5
```

PDF saved to `reports/report_username_timestamp.pdf`

### 6. Use the Voice Agent

```bash
# Text mode (no microphone needed)
python voice_agent/voice_agent.py --text

# Voice mode
python voice_agent/voice_agent.py --voice
```

```
You: What are codewithVamshi5's biggest problems?

RepoGuardian: codewithVamshi5 has submitted 1 PR with a score of 30 out of 100.
Their biggest problem is logic errors appearing 40 times, mainly missing
exception handlers in the orchestrator file. They also have 3 critical
security issues including SQL injection. Recommend fixing the security
issues first before the next PR.
```

---

## 🌐 API Reference

The FastAPI backend exposes these endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/dashboard` | All metrics for the main dashboard |
| `GET` | `/api/users` | List all tracked developers |
| `GET` | `/api/user/{username}` | Full profile for one developer |
| `POST` | `/api/chat` | Chat with the AI about your repo |
| `GET` | `/api/health` | Health check |

### Example: Dashboard response shape

```json
{
  "pull_requests": [...],
  "health_trend":  [{"name": "PR-1", "score": 30}],
  "findings":      [...],
  "agents_data":   [...],
  "live_log":      [...],
  "stats": {
    "total_prs":      1,
    "total_findings": 313,
    "avg_score":      30,
    "critical_count": 84
  }
}
```

---

## 🔧 Configuration

### Rate Limit Strategy (Groq Free Tier)

RepoGuardian is built for the Groq free tier (30 req/min). Agents are staggered automatically:

```
t=0s    PR Review    → up to 3 LLM chunks
t=20s   Security     → 1 LLM call
t=40s   Docs         → 1 LLM call
t=60s   Complexity   → 1 LLM call
        Dependency   → background thread (no LLM)
        Memory       → no LLM
        Auto-Fix     → no LLM
```

Total: ~6 LLM calls spread over 60 seconds — well within limits.

### Switch LLM Provider

```env
# Groq (default — free, fast)
LLM_MODEL=groq
GROQ_API_KEY=gsk_...

# Grok (xAI)
LLM_MODEL=grok
XAI_API_KEY=xai_...

# Anthropic
LLM_MODEL=anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Ollama (local)
LLM_MODEL=ollama
```

---

## 📦 Install Dependencies

```bash
# Core
pip install PyGithub groq python-dotenv fastapi uvicorn

# Analysis tools
pip install pip-audit radon bandit semgrep

# PDF reports
pip install reportlab matplotlib

# Voice agent
pip install SpeechRecognition pyaudio gtts pygame

# Dashboard
cd dashboard && npm install
```

Or install everything at once:

```bash
pip install -r requirements.txt
```

---

## 🗺️ Roadmap

- [ ] Webhook endpoint (auto-trigger on PR open/update)
- [ ] GitHub Actions integration
- [ ] Slack / Teams notifications
- [ ] Multi-repo support
- [ ] Historical trend comparison across developers
- [ ] Team leaderboard on dashboard
- [ ] Custom rule configuration per repo

---

## 🏆 Built At

**KLH Hackathon 2025** — Multi-Agent Intelligence Track

> RepoGuardian was designed and built in 24 hours as a full-stack autonomous code review system demonstrating multi-agent AI coordination, real-time dashboards, developer memory tracking, and voice-powered analytics.

### Team

| Role | Responsibility |
|---|---|
| **P1 — Agents** | `orchestrator.py`, `pr_review.py` |
| **P2 — Tools** | `security.py`, `dependency.py`, `docs.py`, `complexity.py` |
| **P3 — GitHub** | `receiver.py`, webhook integration, commenter |
| **P4 — Dashboard** | React dashboard, charts, voice agent UI |

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

<div align="center">

Made with ❤️ by the RepoGuardian team · KLH Hackathon 2025

**[⭐ Star this repo](https://github.com/codewithVamshi5/repo_guardian)** if RepoGuardian helped you ship better code!

</div>
