# Feature Plan: Custom Prompts on Multi-Email Selection

## Summary

Extend the context menu so that custom prompts (and optionally built-in prompts) can be run against a selection of multiple emails in the message list, not just a single message. The three existing special prompts (Summarize, Spam Filter, Add Tags) already support multi-email selection via `processEmails()` — this feature brings the same capability to arbitrary user-defined prompts.

---

## Thunderbird API Support

Fully supported. The `menus.onClicked` listener receives `info.selectedMessages` — a `MessageList` containing all messages the user had selected when they right-clicked. This is already used today for the special prompts:

```javascript
// Already in mzta-background.js
browser.menus.onClicked.addListener((info, tab) => {
    const promptId = menuItemId.replace('mzta-ctx-', '');
    if (specialContextMenuActions[promptId]) {
        specialContextMenuActions[promptId](getMessages(info.selectedMessages));
    }
});
```

No new Thunderbird API surface is required. The `http://*/*` optional permission already covers any outbound fetch needed.

---

## Design Decisions Required

### 1. Per-message or aggregated?

**Per-message**: run the prompt independently on each selected email. One AI call per message, results delivered separately (one webchat window per message, or queued sequentially).
- Good for: "classify each of these", "draft a reply to each"
- Downside: noisy with large selections; multiple windows

**Aggregated**: concatenate all selected emails into a single prompt using a template+separator pattern (mirrors how multi-email Summarize works today). One AI call, one result.
- Good for: "find the common thread", "compare these quotes", "summarize this thread"
- Downside: prompt can get very long; not suitable for per-message output actions

**Recommendation**: implement per-message first (simpler, more general), with aggregated as a follow-up option. Add a `multi_message_mode` property to the prompt: `"per_message"` or `"aggregated"`.

### 2. Output destination

- **Webchat window** (current behavior for regular prompts) — works for both modes, consistent UX
- **Inline in message pane** — only meaningful for per-message; requires the message to be open
- **New results panel** — a single window listing all results — most useful for aggregated mode

**Recommendation**: use the existing webchat window for the initial implementation. For per-message mode, open one window and stream results sequentially with clear message separators.

### 3. Which prompts are eligible

Not all prompts make sense on a multi-email selection:
- `type: "2"` (compose-only) prompts are never shown in the context menu already — no change needed
- `action: "2"` (substitute text in compose) makes no sense for multi-email — should be excluded
- `need_selected: "1"` prompts that require a text selection within a message are ambiguous — exclude or treat as "use full body"

A new boolean property `multi_message: true` on the prompt definition opts it in. The context menu item is only shown for multi-email selections when this flag is set.

---

## Implementation Scope

### Files to change

| File | Change |
|---|---|
| `js/mzta-prompts.js` | Add `multi_message` property (default `false`) to prompt schema |
| `pages/customprompts/` | Add "Run on multiple emails" checkbox in the custom prompt editor UI |
| `_locales/en/messages.json` | New label for the checkbox |
| `options/mzta-options-default.js` | No change needed (prompt properties are stored per-prompt, not in prefs) |
| `mzta-background.js` | In `menus.onClicked`: detect multi-message selection + `multi_message` prompt → route to new `processEmailsWithPrompt()` |
| `mzta-background.js` | New `processEmailsWithPrompt(messages, prompt)` function |
| `js/mzta-menus.js` | `loadContextMenus()` already puts all eligible prompts in `message_list` context — no structural change, but may need to conditionally show/hide items based on selection count via `onShown` |

### New function: `processEmailsWithPrompt(messages, prompt)`

Mirrors the existing `processEmails()` loop. For each message:

1. Fetch full message body via `browser.messages.getFull(message.id)`
2. Build prompt via `taPromptUtils.preparePrompt()` with the message data
3. Call `mzta_specialCommand` (Web Worker) with the configured API
4. Collect result and pass to `openChatGPT()` or a new aggregated results handler

The existing `processEmails()` function is the direct template — reuse `getMailBody()`, `taPromptUtils.preparePrompt()`, and the worker pattern.

### Dynamic menu visibility (optional enhancement)

Use `browser.menus.onShown` to update menu item visibility based on how many messages are selected:

```javascript
browser.menus.onShown.addListener((info, tab) => {
    const count = info.selectedMessages?.messages?.length ?? 0;
    // show/hide multi_message items based on count
    browser.menus.refresh();
});
```

This requires the `menus` permission (already present).

---

## Constraints

- The **popup menu** (toolbar button) fires in the context of a single active tab/message only. Multi-email selection is only available from the **context menu** (`message_list` context). This is consistent with existing batch features and is the correct UX scope.
- `chatgpt_web` connection type cannot be used for multi-email processing (no headless mode). Prompts with `multi_message: true` should validate that the configured provider is an API type, not `chatgpt_web`.
- Very large selections will produce very long prompts. The existing `max_prompt_length` pref (default 30,000 chars) applies per-message in per-message mode. In aggregated mode, a separate limit or a warning should be considered.

---

## Estimated Effort

Medium — approximately 2–4 days of focused work. The Thunderbird API is fully capable, the multi-message data is already available in the click handler, and `processEmails()` provides a solid blueprint. The main effort is in the new execution path and the custom prompt editor UI change.

---

## Future Extensions

- Aggregated mode with template+separator (mirrors existing multi-email Summarize)
- A dedicated results panel for batch output (list of message → result pairs)
- Progress indicator for long batches (reuse `taWorkingStatus`)
- Per-message caching of results (extend `taStorage` schema)
