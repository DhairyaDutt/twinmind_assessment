# TwinMind — Live Suggestions

A single-page web app that listens to your microphone and continuously surfaces 3 useful, context-aware suggestions while a conversation is happening. Click any suggestion for a detailed answer you can use immediately.

**[Live Demo →](https://quiet-treacle-458547.netlify.app/)**

---

## Setup

1. Get a free [Groq API key](https://console.groq.com)
2. Open the app in Chrome
3. Paste your key when prompted
4. Click **Start** and begin speaking

No install. No build step. No backend.

---

## Stack

| Concern | Choice |
|---|---|
| Transcription | Groq Whisper Large V3 |
| Suggestions + Chat | Groq `openai/gpt-oss-120b` |
| Frontend | Vanilla HTML/CSS/JS, single file |
| Deployment | Netlify (drag and drop) |

Single file was a deliberate choice. The assignment has no backend, no auth, and no shared state — splitting into multiple files would add complexity with no benefit. The file is organized into clearly labeled sections (STATE, SETTINGS, MIC, TRANSCRIPTION, SUGGESTIONS, CHAT, EXPORT) that are easy to navigate.

---

## Prompt Strategy

This is the core of the assignment. Three prompts, each with a specific job.

### 1. Live Suggestions Prompt

The biggest design decision was making the model **phase-aware** before generating suggestions. The prompt instructs the model to silently classify the conversation into one of four phases before picking suggestion types:

- **OPENING** — favor Questions that surface the agenda
- **EXPLORATION** — mix Questions, Talking Points, Answers
- **DECISION** — favor Fact-Checks and Clarifications with data
- **CLOSING** — favor Clarifications (who owns what?) and catch-all Questions

Five suggestion types are available: `QUESTION`, `TALKING_POINT`, `ANSWER`, `FACT_CHECK`, and `CLARIFICATION`. The model picks the right mix for the moment rather than always returning the same pattern.

To prevent stale suggestions, **the previous batch is injected into every prompt** with a hard instruction not to repeat or rephrase those ideas. This keeps suggestions fresh even when the conversation hasn't moved much.

`response_format: { type: "json_object" }` is set at the API level, so JSON is guaranteed — no regex cleanup needed.

Context window: **3,000 chars** of recent transcript. Enough to capture the last few minutes without drowning the model in old context.

### 2. Detailed Answer Prompt (on card click)

The key insight here: a meeting copilot answer should be something you can **act on immediately**, not just read. The prompt enforces a fixed three-part structure:

- **Context** — why this matters right now
- **Key points** — 2-4 specific bullets
- **You could say:** — 1-2 sentences the user can speak out loud, verbatim

The "You could say" section is the most important part and the prompt says so explicitly. It turns the chat panel into a teleprompter, not just a fact sheet.

Context window: **8,000 chars** — the full session transcript so the answer is grounded in everything said so far.

### 3. Chat Prompt

Straightforward. Full transcript injected as system context, full chat history passed as message turns so the model can refer back to earlier exchanges. Temperature 0.5 (lower than suggestions) for more reliable, factual answers.

---

## Tradeoffs

**30-second chunks vs. continuous streaming**
Whisper gets a clean, well-formed audio blob every 30 seconds rather than fragmented real-time chunks. This avoids malformed audio errors and produces more accurate transcripts. The tradeoff is ~30s of latency before the first suggestions appear. Acceptable for a meeting copilot; would reconsider for a real-time dictation tool.

**3,000 char suggestion context vs. full transcript**
Suggestions use only recent transcript to stay relevant to what's happening *now*. Using the full transcript risks the model fixating on something said 20 minutes ago. Detailed answers use the full transcript because you want the complete picture when you click through.

**Single suggestion prompt vs. per-phase prompts**
One prompt handles all four conversation phases via in-prompt instructions rather than switching between four different prompts. Simpler to tune, easier to expose in Settings. The tradeoff is slightly less control per phase, but in practice the phase guidance is specific enough to produce good results.

**No persistence**
Session only — nothing written to disk or localStorage. Clean slate on every reload, which is fine for a meeting tool where each session is independent.

---

## Settings

All prompts and parameters are editable at runtime via the Settings modal. Defaults are optimized for general business meetings in English.

| Setting | Default | What it controls |
|---|---|---|
| Suggestion context | 3,000 chars | How much recent transcript feeds suggestion generation |
| Answer context | 8,000 chars | How much transcript feeds detailed on-click answers |
| Refresh interval | 30s | How often suggestions auto-regenerate |
| All three prompts | See source | Fully editable without touching code |