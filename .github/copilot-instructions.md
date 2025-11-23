<!-- Copilot / AI agent instructions for contributors and coding agents -->
# Repository Overview & agent guidance

This repository implements a Chromium extension (sidebar) that extracts page content with Readability, summarizes it via the Gemini API, and provides an interactive chat UI.

- **Key files:** `manifest.json`, `background.js`, `content.js`, `readability.js`, `sidebar.html`, `sidebar.js`, `prompts.js`, `styles.css`.
- **Runtime model:** Content extraction runs in `content.js` (injected as a content script). The UI runs in `sidebar.html` / `sidebar.js` (loaded inside an iframe). Background API calls to Gemini run in `background.js` (service worker).

**Big picture (data flow & integration points)**
- User action (toolbar or action icon) → `content.js` toggles iframe or shows selection toolbar.
- Selection toolbar posts a `window.postMessage` to the iframe (sidebar) with `{ action: 'performSelectionAction', task, selectedText }`.
- Sidebar (`sidebar.js`) calls `chrome.runtime.sendMessage` to: 1) request page content (`{action: 'getPageContent'}`) which background forwards to content script `extractContent`, and 2) call Gemini (`{action: 'callGeminiAPI', prompt, apiKey, model}`). The background service worker performs fetch to the Gemini REST endpoint.
- Prompts are built in `prompts.js` (see `getSummaryPrompt`) and expect a JSON object back from the model for structured summaries. `sidebar.js` uses `renderApiResponse` to parse JSON (it tolerates code fences like ```json).

**Project-specific patterns & conventions**
- API key & settings: stored via `chrome.storage.sync` — `sidebar.js` reads/writes `apiKey`, `model`, `summaryStrength`, and `theme`.
- Panel UI state & per-tab chat: `sidebar.js` keeps an in-memory `tabChats` Map and persists only when switching tabs; width is saved to `chrome.storage.local` by `content.js` (`geminiPanelWidth`).
- Background responses follow the `{ success: boolean, data?, error? }` convention. Sidebar expects this shape when calling `chrome.runtime.sendMessage`.
- Prompt/output handling: the summarization prompt explicitly requests a JSON response. The model may still return fenced JSON or plain text — `renderApiResponse` has logic to handle both. When changing prompts, update `prompts.js` and verify `renderApiResponse` still parses output correctly.

**Where to edit for common tasks**
- Change summarization wording/JSON schema: edit `prompts.js` (`getSummaryPrompt`).
- Adjust extraction heuristics: edit `readability.js` (third-party code, large file) or `content.js` wrapper that calls it (`extractPageContent`).
- Change Gemini API call behavior (headers, model endpoint): edit `background.js` in `callGeminiAPI`.
- UI/UX changes: `sidebar.html`, `sidebar.js`, and `styles.css`.

**Debugging & developer workflows**
- No build step — install as an unpacked extension: open `chrome://extensions/` → Enable Developer mode → Load unpacked → point to repo folder.
- To inspect the sidebar iframe: open page devtools, switch to the iframe context (or open the extension iframe via the page's DevTools). To inspect the background service worker logs, go to `chrome://extensions/` → Inspect Service Worker for this extension.
- To reload changes: update files then click the refresh button on the extension card in `chrome://extensions/` (service worker will restart).

**Coding-agent guidance (what you can safely change & tests to run locally)**
- You can modify prompts and UI without changing manifest. For API changes, update `background.js` fetch logic and adapt `sidebar.js` parsing accordingly.
- Be conservative editing `readability.js`: it's a vendored/third-party library; prefer small wrapper changes in `content.js` unless you have a clear reason.
- When adding messaging channels between content/sidebar/background, follow the existing conventions: `action` strings and response object `{ success, data, error }`.

**Examples & gotchas (concrete snippets)**
- Request page content (sidebar → background → content script):
  - Sidebar sends: `chrome.runtime.sendMessage({ action: 'getPageContent' }, callback)`
  - Background forwards: `chrome.tabs.sendMessage(tab.id, { action: 'extractContent' }, response)`
  - Content script returns: `{ success: true, content: { title, content } }`
- Gemini call (sidebar → background): `chrome.runtime.sendMessage({ action: 'callGeminiAPI', prompt, apiKey, model, requestJson })` and expect `{ success: true, data: string }`.

**Priority checks before a PR**
- Ensure `prompts.js` JSON schema matches parsing logic in `sidebar.js` (`renderApiResponse`).
- If you change storage keys, update both `sidebar.js` and `content.js` where keys like `geminiPanelWidth` or `apiKey` are read/written.
- Verify cross-origin iframe loading (`web_accessible_resources` in `manifest.json`) if you add new assets referenced by `sidebar.html`.

If any of these sections are unclear or you'd like more detail (examples of common edits, testing checklist, or live debug steps), tell me which area to expand. I'll iterate.
