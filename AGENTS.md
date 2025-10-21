# Agents & Responsibilities

This document defines the autonomous agent network for the unified messaging platform that connects multiple communication channels (WhatsApp, Instagram DM, Messenger, Telegram, TikTok) to a local LLM (ChatGPT API) and routes messages between them.

---

## 1) Router Agent (Backend)
- Endpoint: `POST /internal/events/new_message`
- Accepts normalized message events from all workers.
- Enriches events with channel metadata and stores in SQLite (`threads`, `messages`, `outbox` tables).
- Evaluates routing policy via **Policy Agent** to determine:
  - `AUTO_REPLY`: directly forward to LLM
  - `HUMAN_REVIEW`: send to dashboard for approval
  - `TEMPLATE_REQUIRED`: special case for Meta channels (24h+)
- On auto mode → sends message to **LLM Agent**, then queues the response into Outbox.

---

## 2) Policy Agent
- Maintains compliance with channel-specific rules.
- Rules enforced:
  - WhatsApp, Messenger, IG DM → 24-hour customer-service window
  - After 24h → template or approved tag required
  - Telegram, TikTok → unrestricted
- Tracks last incoming timestamp per thread in DB.
- Returns:
  - `ALLOW`, `TEMPLATE_REQUIRED`, or `HUMAN_REVIEW`.

---

## 3) LLM Agent (ChatGPT)
- Endpoint: `POST /llm/reply`
- Model: `gpt-4o-mini` (configurable via `MODEL` env)
- Uses `OPENAI_API_KEY` for all completions.
- Receives message payload + metadata (channel, language, history context).
- Returns a concise, safe reply respecting channel rules.
- Rejects unsafe or sensitive topics → flagged for human review.

---

## 4) Outbox Agent
- Outgoing message queue per channel.
- API:
  - `GET /internal/outbox/next?channel=...` → returns `{thread_id, text}` or `204`.
- Workers poll this endpoint for new messages to send.
- Logs successful send events to `outbox_log` table and daily JSONL logs.

---

## 5) Worker Agents (Selenium Simulation Layer)
Each worker simulates human messaging through the channel’s web interface.  
No official API keys are used — sessions are authenticated once via QR or login and persisted.

### Common behavior:
- Monitor unread conversations.
- Normalize new messages → `POST /internal/events/new_message`.
- Poll Outbox for pending replies.
- Open corresponding chat via search box.
- Type and send reply text (simulate keystrokes).
- Log every event under `logs/YYYY-MM-DD/app.log`.

### Workers:
| Worker | URL | Note |
|:-------|:----|:-----|
| `whatsapp.py` | https://web.whatsapp.com/ | QR-based login |
| `instagram.py` | https://www.instagram.com/direct/inbox/ | Requires IG Web login |
| `messenger.py` | https://www.messenger.com/ | Page or user login |
| `telegram.py` | https://web.telegram.org/ | Uses WebK |
| `tiktok.py` | https://www.tiktok.com/messages | Business or creator account |

Each keeps its own `PROFILE_DIR` for persistent sessions.

---

## 6) UI Agent (Unified Inbox)
- Built with React + Vite frontend.
- Displays all threads from SQLite in real time.
- Sections:
  - Unified Inbox (all channels)
  - Pending approvals (human-reviewed replies)
  - SEO & Scheduler tabs
  - Live log viewer (SSE or polling)
- Manual overrides push messages back into Outbox.

---

## 7) SEO Agent
- Analyzes outgoing messages or post content.
- Suggests 3 optimized title & description variants.
- Endpoint: `POST /seo/suggest`
- Uses ChatGPT to improve discoverability for social posts.

---

## 8) Scheduler Agent
- Handles scheduled content publication (e.g., Reels, Stories, posts).
- Uses cron or BullMQ job queue.
- At scheduled times, issues commands to corresponding Workers (Selenium opens composer, attaches media, sends).
- Uses same Outbox transport for execution.

---

## 9) Logging & Observability
- Logs saved per day: `logs/YYYY-MM-DD/app.log`
- JSONL schema: `{timestamp, level, event, channel, thread_id, message, notes}`
- Workers send heartbeats every 30 seconds.
- Optional endpoints:
  - `/health` → returns `{ok: true}`
  - `/metrics` → Prometheus-compatible counters.

---

## 10) Database Schema (SQLite)
### Tables:
**threads**
| id | channel | thread_id | last_msg_ts |

**messages**
| id | channel | thread_id | sender_id | text | ts | direction |

**outbox**
| id | channel | thread_id | text | created_at |

All persisted to `data.sqlite` in project root.

---

## 11) Local Development Mode (no API keys)
- Workers operate Selenium-controlled browser tabs instead of APIs.
- Browser sessions stay open; agents mimic human typing.
- Backend still uses ChatGPT key for generating responses.
- No Meta/TikTok/Telegram official API calls are made.
- Logs simulate webhook flow for debugging.

---

## 12) Boot Sequence
1. Launch backend (`node server.js` or `ts-node src/index.ts`)
2. Launch workers (`python workers/whatsapp.py`, etc.)
3. Each worker connects to `/internal/events/new_message` and `/internal/outbox/next`.
4. Messages start flowing → LLM replies → responses sent back.
5. Unified Inbox frontend displays conversation flow.

---

## 13) Folder Layout

```
multichannel-ai-inbox/
│
├─ backend/
│   ├─ src/
│   │   ├─ routes/llm.ts
│   │   ├─ routes/internal.ts
│   │   ├─ services/openai.ts
│   │   ├─ services/store.ts
│   │   └─ index.ts
│   └─ data.sqlite
│
├─ workers/
│   ├─ whatsapp.py
│   ├─ instagram.py
│   ├─ messenger.py
│   ├─ telegram.py
│   └─ tiktok.py
│
├─ frontend/
│   └─ (React/Vite UI)
│
├─ logs/
│   └─ YYYY-MM-DD/
│
├─ .env
└─ AGENTS.md
```

---

## 14) Environment Variables
```
PORT=8080
PUBLIC_URL=http://localhost:8080
OPENAI_API_KEY=sk-yourkey
MODEL=gpt-4o-mini
LOG_ROOT=./logs
BACKEND_URL=http://localhost:8080
PROFILE_DIR=./profiles
```

---

## 15) Security & Recovery
- All tokens/envs stored locally — no external API calls beyond OpenAI.
- Each worker keeps separate Chrome profile; if broken, delete and re-login.
- Logs timestamped to prevent overlap.
- Human review queue to stop unsafe or misrouted responses.

---

## 16) Future Expansion
- Add voice message transcription → text → LLM → reply.
- Add image recognition for media replies.
- Upgrade SQLite → PostgreSQL for concurrent writes.
- Add analytics dashboard (volume, response latency, AI vs human rate).

---

**End of Document**
