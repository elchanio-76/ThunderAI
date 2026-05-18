# Feature Plan: External Integrations (Obsidian, Webhooks, etc.)

## Summary

Add a new output action type that sends AI-generated results to an external application or service after processing. The primary use case is writing summaries, extracted data, or structured notes to an Obsidian vault, but the architecture should be generic enough to support any HTTP endpoint (webhooks, n8n, custom agents, etc.).

**Example workflow**: Order confirmation email arrives → ThunderAI extracts retailer, item, date, cost → writes a structured note to Obsidian → a financial agent picks it up and updates a budget CSV.

---

## Thunderbird API Support

Fully supported via standard `fetch()`. No new Thunderbird API surface is required.

- `http://*/*` and `https://*/*` are already in `optional_permissions` in `manifest.json` (used for Ollama and OpenAI-compatible endpoints)
- The `downloads` permission is already present (used for export features)
- `browser.runtime.connectNative()` is available for native messaging if a filesystem bridge is needed

The extension can make outbound HTTP calls to `localhost` or any remote endpoint from the background script or a Web Worker, using the exact same `fetch()` pattern already used for all API providers.

---

## Integration Approaches

### Option 1: Obsidian Local REST API (recommended)

The [obsidian-local-rest-api](https://github.com/coddingtonbear/obsidian-local-rest-api) community plugin runs a local HTTPS server inside Obsidian (default port 27124, optional HTTP on 27123). It exposes full CRUD on the vault over HTTP with Bearer token authentication.

From ThunderAI's perspective this is identical to calling Ollama — a `fetch()` to `localhost`.

**Key endpoints:**

| Operation | Method | Endpoint |
|---|---|---|
| Create/overwrite note | PUT | `/vault/{path}` |
| Append to note | PATCH + `Operation: append` header | `/vault/{path}` |
| Append to heading | PATCH + `Target-Type: heading` + `Target: {heading}` | `/vault/{path}` |
| Create daily note | POST | `/periodic/daily/` |
| Append to daily note | PATCH | `/periodic/daily/` |
| Search vault | POST | `/search/simple/` |

**Example — append summary to today's daily note under a heading:**

```javascript
await fetch('http://127.0.0.1:27123/periodic/daily/', {
  method: 'PATCH',
  headers: {
    'Authorization': 'Bearer <api-key>',
    'Content-Type': 'text/markdown',
    'Operation': 'append',
    'Target-Type': 'heading',
    'Target': 'Email Summaries',
  },
  body: `\n## ${emailSubject}\n*${emailDate} — ${emailAuthor}*\n\n${summaryText}\n`
});
```

**Certificate friction**: Obsidian's REST API uses a self-signed certificate on the HTTPS port. Thunderbird's `fetch()` will reject it. Solutions:
1. Use the plain HTTP endpoint on port 27123 (user enables it in Obsidian settings — one-time setup)
2. User downloads and trusts the cert in their OS certificate store
3. ThunderAI settings offer a "skip TLS verification" toggle (not ideal for security but pragmatic for localhost)

**Recommendation**: default to HTTP (port 27123) with a note in the UI that HTTPS requires trusting the cert.

### Option 2: Generic HTTP Webhook

A configurable HTTP POST to any URL with a JSON payload. Covers n8n, Make (Integromat), Zapier webhooks, custom agents, Home Assistant, etc.

```javascript
await fetch(webhookUrl, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', ...customHeaders },
  body: JSON.stringify({
    subject: emailSubject,
    author: emailAuthor,
    date: emailDate,
    result: aiResult,
    prompt_id: promptId,
  })
});
```

This is the most general form and should be the underlying primitive that the Obsidian integration is built on top of.

### Option 3: Native Messaging (filesystem bridge)

`browser.runtime.connectNative()` connects to a registered native host program (Python/Node.js script) that has full filesystem access. This allows writing directly to vault markdown files without the Obsidian REST API plugin.

- Requires: a small host script + a JSON manifest registered in the OS (per-platform paths)
- More setup friction; harder to distribute
- Right choice if the user wants zero extra Obsidian plugins and direct file writes
- Not recommended as the primary path — too much user-side setup

### Option 4: `browser.downloads` API (limited)

`browser.downloads.download()` can write a file, but only relative to the browser's default download directory. Cannot write to an arbitrary vault path. Useful only for "export as markdown file" — not for seamless vault integration.

---

## Recommended Architecture

Build a **generic external action system** with Obsidian as the first named integration:

```
AI result arrives in background
  ↓
if prompt has action = "external" (new action type 3)
  ↓
sendToExternal(result, prompt.external_config)
  ↓
  ├── type: "obsidian_rest"  → formats as markdown, calls Obsidian REST API
  ├── type: "webhook"        → POSTs JSON payload to configured URL
  └── type: "native"         → sends via connectNative() to host script
```

---

## Implementation Scope

### New prompt action type

Add `action: 3` ("send to external") to the prompt schema alongside the existing:
- `0` = close
- `1` = reply
- `2` = substitute text
- `3` = send to external (new)

### New prompt properties for external config

```javascript
{
  action: "3",
  external_type: "obsidian_rest" | "webhook" | "native",
  external_url: "http://127.0.0.1:27123",       // base URL
  external_api_key: "",                           // Bearer token
  external_note_path: "Daily/{%mail_date%}.md",  // supports placeholders
  external_heading: "Email Summaries",            // target heading (Obsidian)
  external_operation: "append" | "prepend" | "overwrite",
  external_payload_template: "",                  // JSON template for webhooks
}
```

Note: `external_note_path` supports the existing `{%placeholder%}` syntax, enabling dynamic routing like `Purchases/{%mail_date%}.md` or `Contacts/{%mail_author%}.md`.

### Files to change

| File | Change |
|---|---|
| `js/mzta-prompts.js` | Add new properties to prompt schema |
| `pages/customprompts/` | Add "External action" section in custom prompt editor: type selector, URL, API key, path, heading, operation |
| `_locales/en/messages.json` | New labels for all external config UI fields |
| `options/mzta-options-default.js` | Add global external integration defaults (optional — per-prompt config may be sufficient) |
| `mzta-background.js` | In `openChatGPT()` result handler: if `action === 3`, call `sendToExternal()` instead of/in addition to displaying result |
| `js/mzta-external.js` | **New module** — handles all external dispatch logic (see below) |

### New module: `js/mzta-external.js`

```javascript
export const taExternal = {

  async send(result, config, placeholderValues) {
    switch (config.external_type) {
      case 'obsidian_rest':
        return this.sendToObsidian(result, config, placeholderValues);
      case 'webhook':
        return this.sendWebhook(result, config, placeholderValues);
    }
  },

  async sendToObsidian(result, config, placeholderValues) {
    const notePath = resolvePlaceholders(config.external_note_path, placeholderValues);
    const url = `${config.external_url}/vault/${encodeURIComponent(notePath)}`;
    const headers = {
      'Authorization': `Bearer ${config.external_api_key}`,
      'Content-Type': 'text/markdown',
      'Operation': config.external_operation || 'append',
    };
    if (config.external_heading) {
      headers['Target-Type'] = 'heading';
      headers['Target'] = config.external_heading;
    }
    return fetch(url, { method: 'PATCH', headers, body: result });
  },

  async sendWebhook(result, config, placeholderValues) {
    const payload = buildWebhookPayload(result, config, placeholderValues);
    return fetch(config.external_url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(config.external_api_key ? { 'Authorization': `Bearer ${config.external_api_key}` } : {})
      },
      body: JSON.stringify(payload)
    });
  }
};
```

### Output formatting

The AI result sent to Obsidian should be formatted as clean markdown. The existing `cleanSummaryText()` function in `mzta-background.js` (used for webchat summary saving) is a good starting point. A configurable template per prompt would allow the user to control the note structure:

```
## {%mail_subject%}
*From: {%mail_author%} — {%mail_date%}*

{AI_RESULT}
```

---

## End-to-End Example: Purchase Tracking

**Prompt configuration:**
- Name: "Log Purchase"
- Type: reading email only (`type: 1`)
- Prompt text: `Extract from this email: retailer name, item description, purchase date, and total cost. Return as JSON: {"retailer":"...","item":"...","date":"...","cost":"..."}\n\n{%mail_text_body%}`
- Action: External (`action: 3`)
- External type: Obsidian REST API
- URL: `http://127.0.0.1:27123`
- Note path: `Finance/Purchases/{%mail_date%}.md`
- Heading: `Purchases`
- Operation: append

**Flow:**
1. User right-clicks a purchase confirmation email → "ThunderAI → Log Purchase"
2. ThunderAI sends the prompt to the configured AI provider
3. AI returns JSON: `{"retailer":"Amazon","item":"USB Hub","date":"2026-05-17","cost":"$29.99"}`
4. `mzta-external.js` formats it as markdown and PATCHes the Obsidian daily note
5. Obsidian note now has the entry under the "Purchases" heading
6. Downstream financial agent (Claude via MCP, n8n, custom script) reads the vault via the same REST API or the built-in MCP server at `http://127.0.0.1:27123/mcp/`

---

## Constraints

- `chatgpt_web` connection type cannot be used for prompts with `action: 3` — the webchat window flow doesn't have a clean result-capture hook. Validate and show an error if configured.
- The Obsidian REST API plugin must be running (Obsidian must be open). ThunderAI should handle connection errors gracefully and show a clear message.
- API keys are stored in `browser.storage.local` — same security model as existing API keys (OpenAI, Anthropic, etc.). Not encrypted at rest, but not exposed to web content.
- The `external_api_key` field must never appear in logs. Follow the existing pattern in `mzta-menus.js` which redacts `_api_key` fields in debug output.

---

## Estimated Effort

Medium-to-large — approximately 4–7 days of focused work:
- New module `mzta-external.js`: 1 day
- Custom prompt editor UI additions: 1–2 days
- Background integration + error handling: 1 day
- Obsidian-specific formatting + testing: 1 day
- Webhook support + payload templating: 1 day

---

## Future Extensions

- **Global external integrations** — a shared integration config (like the existing API provider config) that individual prompts can reference by name, rather than duplicating URL/key per prompt
- **Native messaging bridge** — for users who want direct filesystem writes without the Obsidian plugin
- **Result preview** — show the formatted output before sending, with a confirm/cancel step
- **Retry logic** — if the external endpoint is unavailable, queue the result and retry when it comes back online
- **Two-way sync** — read from Obsidian (e.g. fetch context notes before building a prompt) using the same REST API
- **Downstream agent trigger** — after writing to Obsidian, optionally POST to a second webhook to notify the financial agent that new data is available
