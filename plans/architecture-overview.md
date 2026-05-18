# ThunderAI — Architecture & Technology Overview

## What It Is

ThunderAI is a **Thunderbird WebExtension (Manifest V2)** that adds AI-augmented email workflows directly into the Thunderbird UI. It supports six AI providers and a rich set of email-aware features: summarization, translation, spam filtering, auto-tagging, calendar event extraction, task creation, and general-purpose prompting.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Runtime | Thunderbird WebExtension (MV2), plain ES6 modules |
| Language | Vanilla JavaScript — no build tools, no npm, no transpilation |
| Storage | `browser.storage.local` (prefs + per-message data) + `browser.storage.session` (in-flight state) |
| Concurrency | Web Workers (one per API provider) |
| i18n | WebExtension `browser.i18n` API, 16 languages via Weblate |
| UI | Plain HTML/CSS, no framework |
| Markdown rendering | `markdown-it` (vendored) |
| Diff viewing | `diff.js` (vendored, injected into chatgpt.com) |

---

## Cross-Platform Status

Fully cross-platform. Runs identically on Windows, macOS, and Linux. The entire codebase runs inside Thunderbird's JavaScript engine (SpiderMonkey) — no native binaries, no shell commands, no OS-level APIs, no platform detection. The only user-side platform concern is Ollama, which requires a locally running server, but Ollama has native builds for all three platforms.

---

## Execution Contexts

The extension runs across five distinct browser contexts:

```
Background Page   mzta-background.html / mzta-background.js
                  ↑ main orchestrator, always running

Popup             popup/mzta-popup.html / mzta-popup.js
                  ↑ shown on toolbar click or Ctrl+Alt+A

Options Page      options/mzta-options.html / options/mzta-options.js
                  ↑ settings UI

Feature Pages     pages/*/
                  ↑ per-feature settings (summarize, translate, spam, tags, etc.)

Content Script    js/mzta-compose-script.js
                  ↑ injected into every compose and message display tab
                  ↑ renders inline banners (summary, translation, spam, errors)

Web Workers       js/workers/model-worker-*.js
                  ↑ one per API provider, handles HTTP calls off the main thread

API Webchat       api_webchat/
                  ↑ standalone chat window opened for API-based interactions
```

---

## Core Data Flow

### User-triggered prompt

```
User clicks popup / presses Ctrl+Alt+A
  → popup/mzta-popup.js (renders prompt list)
  → sendMessage to mzta-background.js
  → mzta-placeholders.js resolves {%placeholder%} tokens from email data
  → mzta-prompts.js builds final prompt string
  → dispatched to provider:
      chatgpt_web   → mzta-chatgpt.js (opens chatgpt.com, DOM automation)
      all API types → Web Worker → HTTP fetch → result back to background
  → mzta-compose-script.js inserts result into compose/display window
```

### Automatic inline summary/translation (on message open)

```
User opens a message
  → mzta-compose-script.js sends "initSummary" / "initTranslation" to background
  → background checks prefs (auto mode, display mode, cache)
  → cache hit → render immediately
  → cache miss → Web Worker → result → taSummaryStore / taTranslationStore → render banner
```

### Background pre-caching (on email receive)

```
New email arrives → browser.messages.onNewMailReceived
  → processEmails() (shared loop for tags, spam, summary, translation)
  → silent generation via Web Worker, no UI
  → result stored in taStorage keyed by headerMessageId
  → when user opens the message → instant cache hit
```

---

## AI Provider Architecture

Each provider follows the same pattern: an API module + a dedicated Web Worker.

| Provider | `connection_type` | API Module | Worker |
|---|---|---|---|
| ChatGPT Web | `chatgpt_web` | `mzta-chatgpt.js` (DOM automation) | none |
| OpenAI API | `chatgpt_api` | `api/openai_responses.js` | `model-worker-openai_responses.js` |
| Google Gemini | `google_gemini_api` | `api/google_gemini.js` | `model-worker-google_gemini.js` |
| Anthropic/Claude | `anthropic_api` | `api/anthropic.js` | `model-worker-anthropic.js` |
| Ollama | `ollama_api` | `api/ollama.js` | `model-worker-ollama.js` |
| OpenAI-compatible | `openai_comp_api` | `api/openai_comp.js` | `model-worker-openai_comp.js` |

Pre-configured OpenAI-compatible providers (DeepSeek, Grok, Mistral, OpenRouter, Perplexity) are defined in `api/openai_comp_configs.js`.

The global `connection_type` pref selects the default provider. Each of the 6 special prompts can independently override this with its own `{prefix}_connection_type` pref.

**ChatGPT Web** is the only provider that doesn't use a worker — it opens a real browser window to `chatgpt.com`, injects the prompt via DOM automation, and reads back the response. A content script (`js/lib/diff.js`) is injected into ChatGPT pages for diff-view support.

**Thinking/reasoning output** is handled provider-specifically: Anthropic sends thinking tokens as separate SSE events; Ollama/OpenAI-compatible wrap them in `<think>…</think>` tags in the normal stream. Both render as a collapsible `<details>` block in the webchat UI.

---

## Prompt System

Two categories of prompts:

**Built-in prompts** (`js/mzta-prompts.js`) — 8 default prompts (reply, rewrite, proofread, classify, etc.) plus 9 special prompts.

**Special prompts** — trigger Thunderbird-specific actions beyond just sending text to the AI:

| ID | Feature |
|---|---|
| `prompt_add_tags` | Auto-tag the email |
| `prompt_spamfilter` | Spam classification + optional move |
| `prompt_summarize` | Email summarization |
| `prompt_translate_this` | Inline translation |
| `prompt_get_calendar_event` | Extract + create calendar event |
| `prompt_get_task` | Extract + create task |

**Custom prompts** are user-created and stored in `browser.storage.local`.

Each prompt has a rich property set: visibility type (reading/composing/both), action type (close/reply/substitute), per-prompt API override, menu placement, and icon. Menu ordering is position-based with drag-and-drop via `pages/menu_order/`.

---

## Placeholder System

Prompts use `{%placeholder_id%}` tokens that are resolved at runtime from live email data. There are ~28 built-in placeholders covering email body (text/HTML), subject, headers, sender, recipients, tags, identity, folder, attachments, datetime, and ThunderAI-specific values (language, signature). Dynamic placeholders use a colon syntax: `{%mail_headers:x-spam-score%}`, `{%additional_text:my_field%}`.

Custom placeholders (user-defined static text, prefixed `thunderai_custom_`) are stored in `browser.storage.local` and merged at runtime.

---

## Storage Architecture

Three layers:

1. **Preferences** — flat key/value in `browser.storage.local` (and `browser.storage.sync` for some prefs). All keys and defaults defined in `options/mzta-options-default.js`.

2. **Per-message data** — `taStorage` class (`js/mzta-storage.js`), keyed by `msg:<headerMessageId>`. Stores summary, spam report, and translation per message. Schema-versioned with age-based cleanup.

3. **In-flight state** — `browser.storage.session` tracks which messages are currently being processed (prevents duplicate generation).

`taSummaryStore` and `taTranslationStore` wrap `taStorage` with feature-specific logic: 100-entry LRU cache, error state tracking, and processing-state management.

---

## Inline UI Injection

`js/mzta-compose-script.js` is registered as both a compose script and a message display script. It injects a `#mzta-container` div into every message view, which hosts:

- A unified toolbar (spam badge, summary/translation trigger buttons)
- Content panels: summary banner, translation banner (green/teal), spam explanation, generic error panel
- Each panel has refresh (↻) and dismiss (×) controls

The script communicates with the background via `browser.runtime.sendMessage` / `browser.tabs.sendMessage`.

---

## Permissions Model

Core permissions declared in `manifest.json`: `compose`, `messagesRead`, `messagesModify`, `messagesTagsList`, `storage`, `menus`, `tabs`, `activeTab`, `accountsRead`, `downloads`.

Optional permissions (requested at runtime): `addressBooks`, `clipboardRead`, `messagesTags`, `messagesUpdate`, `messagesMove`, and host permissions for each API provider (`https://*.anthropic.com/*`, `https://*.chatgpt.com/*`, `http://*/*`, etc.).

---

## Key Design Decisions

- **No build tooling** — every JS file is a plain ES6 module loaded directly. Keeps the extension reviewable and avoids bundler complexity, at the cost of no tree-shaking or minification.
- **Workers for all API calls** — keeps the Thunderbird UI responsive during potentially slow LLM requests.
- **Per-message caching** — summaries and translations are cached by `headerMessageId`, enabling instant display on re-open and background pre-caching on email receive.
- **Feature flags** — each major feature (summarize, translate, spam, tags, calendar, task) is independently toggled, and each can use a different AI provider.
- **No Experiment APIs** — the extension relies entirely on standard Thunderbird WebExtension APIs, making it maintainable across monthly Thunderbird releases.

---

## Key Module Reference

| File | Role |
|---|---|
| `mzta-background.js` | Main orchestrator: listens for messages, coordinates all features |
| `js/mzta-menus.js` | Context menu and popup menu creation and management |
| `js/mzta-prompts.js` | Prompt definitions (built-in + custom loading) |
| `js/mzta-placeholders.js` | Placeholder definitions and resolution logic |
| `js/mzta-utils.js` | General utilities (email parsing, storage helpers, etc.) |
| `js/mzta-utils-prompt.js` | Prompt-specific utilities (`buildSummaryPrompt`, `buildTranslationPrompt`) |
| `js/mzta-compose-script.js` | Content script: injects AI response and inline banners |
| `js/mzta-chatgpt.js` | ChatGPT Web integration (DOM automation) |
| `js/mzta-special-commands.js` | Handles special prompt actions (tags, calendar, task) |
| `js/mzta-spamreport.js` | Spam filter logic |
| `js/mzta-storage.js` | Unified per-message storage layer (`taStorage`) |
| `js/mzta-summarystore.js` | Summary-specific storage wrapper (`taSummaryStore`) |
| `js/mzta-translationstore.js` | Translation-specific storage wrapper (`taTranslationStore`) |
| `js/mzta-working-status.js` | Visual status indicator during AI processing |
| `options/mzta-options-default.js` | All default preference values and integration config |
| `api_webchat/controller.js` | Webchat window controller (wires worker ↔ UI) |
