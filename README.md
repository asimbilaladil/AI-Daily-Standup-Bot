# 🤖 AI Daily Standup Bot — Build Checklist

> **Stack:** Node.js · Recall.ai · Claude API · ElevenLabs · Microsoft Graph · Redis  
> **Goal:** A bot that joins your recurring Zoom standup, asks each team member personalised questions based on their Excel updates, tracks blockers, and writes results back to the sheet.

---

## Before You Start — Account Setup

Get all accounts and API keys ready before writing a single line of code.

- [ ] Create a [Recall.ai](https://www.recall.ai) account → grab your **API key**
- [ ] Create an [Anthropic](https://console.anthropic.com) account → grab your **API key**
- [ ] Create an [ElevenLabs](https://elevenlabs.io) account → grab your **API key** + pick a **Voice ID**
- [ ] Register an app in [Azure Portal](https://portal.azure.com) for Microsoft Graph access
  - [ ] Set permissions: `Files.ReadWrite`, `Sites.ReadWrite.All`
  - [ ] Copy **Client ID**, **Client Secret**, **Tenant ID**
- [ ] Get your **Excel file ID** from OneDrive (visible in the file's share URL)
- [ ] Create an [Upstash](https://upstash.com) Redis database → grab the **Redis URL**
- [ ] Set up a [Railway](https://railway.app) or [Render](https://render.com) account for hosting

---

## Phase 1 — Project Setup & Zoom Bot Foundation

**Goal:** Bot joins your Zoom meeting and logs transcripts to console. No voice yet.

### 1.1 Project Scaffold

- [ ] Create project folder `zoom-standup-bot/`
- [ ] Run `npm init -y`
- [ ] Install core dependencies:
  ```bash
  npm install express axios dotenv node-cron @anthropic-ai/sdk redis
  ```
- [ ] Create `.env` file with all keys from the Account Setup section above:
  ```
  RECALL_API_KEY=
  ANTHROPIC_API_KEY=
  ELEVENLABS_API_KEY=
  ELEVENLABS_VOICE_ID=
  MS_GRAPH_CLIENT_ID=
  MS_GRAPH_CLIENT_SECRET=
  MS_GRAPH_TENANT_ID=
  EXCEL_FILE_ID=
  REDIS_URL=
  ZOOM_MEETING_LINK=
  STANDUP_CRON=0 9 * * 1-5
  WEBHOOK_SECRET=
  WEBHOOK_URL=https://your-deployed-url.com
  ```
- [ ] Add `.env` to `.gitignore`
- [ ] Create the full folder structure:
  ```
  src/
  ├── index.js
  ├── scheduler.js
  ├── meetingOrchestrator.js
  ├── recallClient.js
  ├── claudeClient.js
  ├── voiceClient.js
  ├── excelClient.js
  ├── memoryStore.js
  ├── contextBuilder.js
  ├── conversationState.js
  ├── webhooks/
  │   └── recallWebhook.js
  ├── config/
  │   └── teamConfig.json
  └── utils/
      ├── silenceDetector.js
      └── blockerExtractor.js
  ```

### 1.2 Team Config

- [ ] Fill in `teamConfig.json` with your team members:
  ```json
  {
    "members": [
      {
        "name": "Ali",
        "role": "Backend Developer",
        "zoomDisplayName": "Ali Khan",
        "excelRow": 2
      }
    ],
    "meetingLink": "https://zoom.us/j/your-link",
    "standupOrder": ["Ali", "Sara", "Omar"]
  }
  ```

### 1.3 Recall.ai — Bot Joins Zoom

- [ ] Build `recallClient.js`:
  - [ ] `joinMeeting(url, botName)` — creates bot via Recall.ai API, returns `botId`
  - [ ] `leaveMeeting(botId)` — removes bot from meeting
  - [ ] `getBotStatus(botId)` — checks if bot is in meeting / done
- [ ] Build `webhooks/recallWebhook.js`:
  - [ ] POST endpoint `/recall/transcript` to receive real-time transcript chunks
  - [ ] Validate webhook secret header
  - [ ] Parse speaker name + transcript text from payload
  - [ ] Log to console for now
- [ ] Set up `index.js` — Express server on port 3000, mount webhook route
- [ ] Deploy to Railway/Render and update `WEBHOOK_URL` in `.env`
- [ ] Set the webhook URL in your Recall.ai dashboard

### 1.4 Scheduler

- [ ] Build `scheduler.js` using `node-cron`
- [ ] Trigger `meetingOrchestrator.js` at the time in `STANDUP_CRON`
- [ ] Add a manual trigger endpoint `POST /trigger-standup` for testing without waiting for cron

### 1.5 Silence Detection

- [ ] Build `utils/silenceDetector.js`
  - [ ] Track timestamp of last transcript chunk per speaker
  - [ ] Fire a "speaker done" event after 2.5 seconds of no new chunks
  - [ ] Expose `onSpeakerFinished(callback)` hook

### ✅ Phase 1 Done When:
- Bot joins your Zoom meeting automatically at scheduled time
- Console logs show who said what in real time
- Bot leaves after a set duration

---

## Phase 2 — AI Voice Layer

**Goal:** Bot speaks in a real voice and asks basic standup questions.

### 2.1 ElevenLabs TTS

- [ ] Build `voiceClient.js`:
  - [ ] `textToSpeech(text)` — calls ElevenLabs API, returns MP3 audio buffer
  - [ ] Handle errors and retry once on failure
- [ ] Test independently: generate a test MP3 and play it locally to confirm voice quality

### 2.2 Audio Injection into Zoom

- [ ] Add `speakInMeeting(botId, audioBuffer)` to `recallClient.js`
  - [ ] Sends base64-encoded MP3 to Recall.ai bot speak endpoint
- [ ] Test: bot joins meeting and says "Hello, this is a test."

### 2.3 Claude — Basic Question Generation

- [ ] Build `claudeClient.js`:
  - [ ] `getNextQuestion(context, memberName, transcript)` — returns spoken text only
  - [ ] `generateOpeningStatement(memberList)` — meeting opener
  - [ ] `generateClosingStatement(meetingSummary)` — meeting closer
- [ ] Write a basic system prompt (no context yet — just role and tone):
  ```
  You are a friendly AI standup facilitator. Ask each team member:
  what they did yesterday, if they have any blockers, and what they
  plan to do today. Be concise — max 2–3 sentences at a time.
  ```

### 2.4 Conversation State Machine

- [ ] Build `conversationState.js` with states:
  - `IDLE` → `OPENING` → `MEMBER_TURN` → `AI_PROCESSING` → `AI_SPEAKING` → `NEXT_MEMBER` → `CLOSING` → `POST_MEETING` → `DONE`
- [ ] Ensure bot never speaks while a member is talking (check state before injecting audio)

### 2.5 Basic Orchestrator

- [ ] Build `meetingOrchestrator.js`:
  - [ ] On trigger: call `recallClient.joinMeeting()`
  - [ ] Wait for bot status = "in_call"
  - [ ] Deliver opening statement via voice
  - [ ] Loop through `standupOrder` from teamConfig
  - [ ] For each member: ask question → wait for silence → get Claude response → speak
  - [ ] Deliver closing statement
  - [ ] Call `recallClient.leaveMeeting()`

### ✅ Phase 2 Done When:
- Bot joins Zoom, speaks an opening, asks each member a question by name, and closes the meeting
- Conversation flows naturally: bot asks → waits → responds → moves on

---

## Phase 3 — Context & Memory

**Goal:** Bot asks personalised questions using Excel data and remembers previous meetings.

### 3.1 Redis Memory Store

- [ ] Build `memoryStore.js`:
  - [ ] `saveMeetingNotes(date, notes)` — stores structured notes keyed by date
  - [ ] `getLastMeetingNotes()` — retrieves most recent meeting summary
  - [ ] `saveBlocker(memberName, blocker)` — adds to open blocker list
  - [ ] `resolveBlocker(memberName, blocker)` — marks blocker as resolved
  - [ ] `getOpenBlockers(memberName)` — returns unresolved blockers for a member
  - [ ] `setSessionState(key, value)` / `getSessionState(key)` — in-meeting turn tracking

### 3.2 Microsoft Graph — Read Excel

- [ ] Install Graph SDK: `npm install @microsoft/microsoft-graph-client`
- [ ] Build `excelClient.js`:
  - [ ] `authenticate()` — gets access token using client credentials flow
  - [ ] `getTeamUpdates()` — reads used range from `Standup` worksheet
  - [ ] `parseExcelRows(values)` — maps rows to member objects:
    ```js
    { name, date, yesterdayLog, todayPlan, blockers, hoursLogged }
    ```
- [ ] Test: print parsed Excel data to console before a mock standup

### 3.3 Context Builder

- [ ] Build `contextBuilder.js`:
  - [ ] `buildMeetingContext()` — assembles full context object:
    - Excel data per member
    - Previous meeting notes from Redis
    - Open blockers from Redis
    - Team profiles from teamConfig.json
  - [ ] `buildMemberContext(memberName)` — focused context for a single member's turn

### 3.4 Upgrade Claude Prompts

- [ ] Update `claudeClient.js` to accept full context in every call
- [ ] Update system prompt to reference Excel data and previous meeting notes:
  ```
  You have full context about this team. Before asking each member,
  you know their Excel log for today, yesterday's stated plan, and any
  open blockers. Reference these naturally in your questions.
  ```
- [ ] Add `extractStructuredUpdate(transcript)` — parses a member's response into:
  ```js
  { completed, blockers, todayPlan, needsHelp }
  ```

### 3.5 Blocker Extractor

- [ ] Build `utils/blockerExtractor.js`:
  - [ ] Scan transcript for blocker keywords: "blocked", "stuck", "waiting on", "can't", "issue with"
  - [ ] Return blocker text if detected, otherwise null
  - [ ] Auto-save detected blockers to Redis via `memoryStore`

### ✅ Phase 3 Done When:
- Bot asks "Ali, I see you logged 6 hours on the payment module yesterday — how did that go?"
- Open blockers from last meeting are referenced naturally
- All responses are saved to Redis after the meeting

---

## Phase 4 — Output & Polish

**Goal:** Excel auto-updates after each meeting. Claude chat context updated. System is reliable.

### 4.1 Microsoft Graph — Write to Excel

- [ ] Add `writeStandupUpdate(memberName, update)` to `excelClient.js`
  - [ ] Finds the correct member row
  - [ ] Appends a new date-stamped entry: completed / today / blockers / hours
- [ ] Call this for every member at the end of `meetingOrchestrator.js`
- [ ] Test: run a mock standup and verify Excel updates correctly

### 4.2 Claude Chat Context Update

- [ ] Add `generateMeetingSummary(allMemberUpdates)` to `claudeClient.js`
  - [ ] Returns a concise paragraph: what the team covered, key blockers, today's targets
- [ ] Save this summary to Redis as `lastMeetingNotes`
- [ ] Optionally: POST summary to a webhook or file so you can paste it into your Claude chat manually (or automate if you use Claude Projects API)

### 4.3 Post-Meeting Report

- [ ] Build a simple post-meeting report object saved to Redis:
  ```js
  {
    date, duration, membersPresent,
    updates: [{ name, completed, todayPlan, blockers }],
    openBlockers: [...],
    summary: "..."
  }
  ```
- [ ] Add `GET /last-meeting` endpoint on your Express server to view the report

### 4.4 Error Handling & Reliability

- [ ] Handle member not present in meeting (skip gracefully, note as absent)
- [ ] Handle Recall.ai webhook timeout — if no transcript after 5 min, close meeting gracefully
- [ ] Handle ElevenLabs API failure — retry once, then skip speaking and log error
- [ ] Handle Excel write failure — log error, save to Redis as pending retry
- [ ] Add try/catch around every external API call
- [ ] Add request logging with timestamps to all major pipeline steps

### 4.5 Final Testing

- [ ] Run a full end-to-end test with your real Zoom recurring link
- [ ] Confirm transcripts are accurate for all team members
- [ ] Confirm Excel updates after meeting with correct data
- [ ] Confirm Redis has structured notes from the meeting
- [ ] Run two consecutive days — confirm Day 2 references Day 1's targets correctly
- [ ] Confirm bot handles a member saying nothing / staying silent

### ✅ Phase 4 Done When:
- Bot runs fully autonomously every weekday
- Excel is updated automatically after every meeting
- Redis memory connects yesterday's targets to today's questions
- System recovers gracefully from API errors

---

## Ongoing Maintenance

- [ ] Monitor Recall.ai changelog for any Zoom SDK updates
- [ ] Review Claude prompts monthly — adjust tone based on team feedback
- [ ] Check Redis memory size monthly — prune notes older than 30 days
- [ ] Review ElevenLabs usage monthly — switch to VAPI if bot speech exceeds 7 min/day consistently
- [ ] Keep `.env` secrets rotated every 90 days

---

## Quick Reference — Key URLs

| Service | Dashboard | Docs |
|---|---|---|
| Recall.ai | [app.recall.ai](https://app.recall.ai) | [docs.recall.ai](https://docs.recall.ai) |
| Anthropic | [console.anthropic.com](https://console.anthropic.com) | [docs.anthropic.com](https://docs.anthropic.com) |
| ElevenLabs | [elevenlabs.io](https://elevenlabs.io) | [docs.elevenlabs.io](https://docs.elevenlabs.io) |
| Microsoft Graph | [portal.azure.com](https://portal.azure.com) | [learn.microsoft.com/graph](https://learn.microsoft.com/en-us/graph/) |
| Upstash Redis | [console.upstash.com](https://console.upstash.com) | [docs.upstash.com](https://docs.upstash.com) |
| Railway (hosting) | [railway.app](https://railway.app) | [docs.railway.app](https://docs.railway.app) |

---

> **Start here → Phase 1, Step 1.1.** Get the bot joining Zoom before anything else. Once transcripts are flowing, every other phase is additive.
