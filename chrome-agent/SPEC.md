# Side Panel Agent — Product & Technical Spec

**Status:** Draft v0.1
**Working name:** Side Panel Agent
**Platform:** Chrome extension (Manifest V3), Chrome 138+

---

## 1. Summary

A Chrome extension that opens a chat in the side panel where the user describes a task in plain language ("unsubscribe me from these newsletters", "add the cheapest 2-pack to my cart", "fill this form with my shipping info"). An agent then **plans, observes the page, and acts on it** — clicking, typing, scrolling, navigating — one step at a time, while showing every step and **pausing for confirmation before anything irreversible**.

It is built only on what ships with Chrome:

- **Built-in AI (Gemini Nano)** through the Prompt API (`LanguageModel`) for reasoning, with on-device inference by default.
- **Extension APIs**: `sidePanel`, `scripting`, `tabs`, `storage`, `activeTab`, and optionally `debugger` (Chrome DevTools Protocol).
- **Vanilla web platform**: ES modules, HTML, CSS. No framework and no bundler are required.

## 2. Goals and non-goals

### Goals
1. Complete short, multi-step browser tasks (≤ 25 actions) on ordinary websites from a natural-language instruction.
2. Be **transparent**: the user always sees the current plan, the step being executed, and the element being acted on.
3. Be **safe by default**: no irreversible action (submit, purchase, send, delete, post) without explicit user confirmation.
4. Be **private by default**: page content and instructions stay on-device unless the user opts into a cloud model.
5. Have **zero build step**: load unpacked from the source folder and it runs.

### Non-goals (v1)
- Running unattended or on a schedule (no background agents while the panel is closed).
- Solving CAPTCHAs, bypassing bot detection, or automating logins/credential entry.
- Cross-browser support (Firefox/Safari).
- Acting inside cross-origin iframes, `chrome://` pages, the Chrome Web Store, PDFs, or other extension pages.
- Long-running research tasks across dozens of tabs.

## 3. Users and use cases

**Primary user:** a non-developer who wants repetitive or fiddly web chores done in their own, signed-in browser.

| # | User story | Example |
|---|---|---|
| U1 | Ask questions about the current page | "What's the return policy on this page?" |
| U2 | Fill forms from saved profile data | "Fill in my shipping address." |
| U3 | Do a short, multi-step flow in one tab | "Find the unsubscribe link and unsubscribe me." |
| U4 | Navigate and compare across a few tabs | "Open the first 3 results and tell me which is cheapest." |
| U5 | Repeat an action over a list | "Archive every email from noreply@ on this page." |
| U6 | Stop or take over at any moment | User clicks **Stop**, or edits a field mid-task. |

## 4. User experience

### 4.1 Entry points
- Toolbar icon → opens the side panel (`chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true })`).
- Keyboard shortcut (`commands`, default `Alt+Shift+A`).
- Context menu on selected text: "Ask agent about this".

### 4.2 Side panel layout
```
┌─────────────────────────────┐
│ ● On-device  · example.com  │  header: model status, active site, ⚙
├─────────────────────────────┤
│ You: unsubscribe me from…   │
│                             │
│ Plan                        │  collapsible, updates as agent re-plans
│  1. ✓ Find unsubscribe link │
│  2. ▶ Open it               │
│  3. ○ Confirm unsubscribe   │
│                             │
│ Step 4 · click "Unsubscribe"│  step log, one row per action
│ ┌─────────────────────────┐ │
│ │ ⚠ This will submit a    │ │  confirmation card (high-risk actions)
│ │ form on example.com     │ │
│ │ [Allow once] [Deny]     │ │
│ └─────────────────────────┘ │
├─────────────────────────────┤
│ [ Type a task…      ] [■ Stop]│
└─────────────────────────────┘
```

### 4.3 On-page feedback
- The element about to be acted on gets a highlight outline and a small label ("Agent: click") for ~600 ms before the action.
- While a task runs, a thin banner sits at the top of the page: "Agent is working — press Esc to stop". Pressing Esc on the page stops the task.

### 4.4 Task lifecycle (user-visible states)
`idle → planning → running ⇄ awaiting_confirmation → (done | failed | stopped)`

`awaiting_user_input` is entered when the agent calls `ask_user` (e.g. "Which size?").

### 4.5 Settings page (`options.html`)
- **Model:** On-device (default) / Cloud (bring your own API key) / Auto (on-device first, cloud when on-device is unavailable or the task fails).
- **Profile data** for form filling: name, email, phone, addresses. Stored in `chrome.storage.local`; never sent to a cloud model unless the user enables that per field.
- **Site permissions:** allowlist / blocklist; "always ask before acting on a new site" (default on).
- **Confirmation policy:** confirm high-risk only (default) / confirm every action / (no "never confirm" option in v1).
- **Limits:** max steps per task (default 25), max tabs the agent may open (default 3).
- **Advanced:** enable DevTools Protocol mode (see §6.4).

## 5. Architecture

### 5.1 Components

```
┌──────────────────────────┐  runtime.connect (port)   ┌─────────────────────────┐
│ Side panel (sidepanel.html)│◄────────────────────────►│ Service worker          │
│  - Chat UI                │                           │  (background.js)        │
│  - Agent loop (agent/*)   │                           │  - panel behavior       │
│  - LanguageModel session  │                           │  - commands, menus      │
│  - Task state             │                           │  - permission requests  │
└────────────┬─────────────┘                           │  - tab/nav events       │
             │ chrome.tabs.sendMessage                  └─────────────────────────┘
             ▼
┌──────────────────────────┐
│ Content script (page.js)  │  injected via chrome.scripting on demand
│  - snapshot()             │
│  - act(action)            │
│  - highlight / banner     │
│  - ref registry           │
└──────────────────────────┘
```

**Where the agent loop lives: the side panel document.** Reasons:
- The built-in AI APIs are exposed to window contexts; their availability in extension service workers is not something to rely on. The side panel is a full window context.
- The panel being open is the product requirement anyway (no unattended runs).
- MV3 service workers are terminated after ~30 s idle, while model inference can take longer. A document is not subject to that.

Task state is mirrored to `chrome.storage.session` after every step, so a reload of the panel can show what happened and offer to **resume** or **discard** the task.

> Verify at M0: confirm `LanguageModel` is available in the side panel on the target Chrome version and hardware. If not, fall back to an offscreen document (`chrome.offscreen`, reason `WORKERS`/`DOM_PARSER`) hosting the session.

### 5.2 File layout
```
chrome-agent/
  manifest.json
  background.js              service worker
  sidepanel/
    sidepanel.html
    sidepanel.css
    sidepanel.js             UI wiring
  agent/
    loop.js                  observe → decide → act loop
    model.js                 provider interface + on-device + cloud providers
    prompt.js                system prompt, message builders
    schema.js                action JSON schema (responseConstraint)
    policy.js                risk classification + confirmation gates
    tools.js                 tool registry: name → executor
    state.js                 Task/Step types, persistence
  content/
    page.js                  snapshot, act, highlight (injected)
  options/
    options.html
    options.js
  test/
    fixtures/                local HTML pages for deterministic tests
    e2e/                     Playwright specs
    unit/                    pure-function tests
  icons/
```
Plain ES modules with JSDoc type annotations; `// @ts-check` at file tops so editors type-check without a compiler.

### 5.3 Manifest (initial)
```json
{
  "manifest_version": 3,
  "name": "Side Panel Agent",
  "version": "0.1.0",
  "minimum_chrome_version": "138",
  "action": { "default_title": "Open agent" },
  "background": { "service_worker": "background.js", "type": "module" },
  "side_panel": { "default_path": "sidepanel/sidepanel.html" },
  "options_page": "options/options.html",
  "permissions": ["sidePanel", "storage", "scripting", "activeTab", "tabs", "contextMenus"],
  "optional_permissions": ["debugger"],
  "optional_host_permissions": ["<all_urls>"],
  "commands": {
    "_execute_action": { "suggested_key": { "default": "Alt+Shift+A" } }
  }
}
```
- `activeTab` covers the tab the user invoked the agent on. Before acting on any other site (navigation, new tabs), the agent requests that origin via `chrome.permissions.request` — which needs a user gesture, so the panel shows an **Allow on site.com** button.
- `debugger` is optional and only requested if the user enables DevTools Protocol mode.

## 6. Observation: how the agent sees the page

### 6.1 Snapshot format
The content script produces a compact, text-first **semantic snapshot**, not raw HTML. It is modeled on an accessibility tree, limited to what matters for acting:

```
url: https://shop.example.com/cart
title: Your cart
viewport: 1-38 of 112 lines (scroll for more)

[r1] heading "Your cart (2 items)"
[r2] link "Continue shopping"
[r3] textbox "Promo code" value=""
[r4] button "Apply"
[r5] list "Cart items"
  [r6] text "Wireless mouse — $24.99"
  [r7] combobox "Quantity" value="1" options=[1,2,3,4,5]
  [r8] button "Remove"
[r9] button "Checkout" ⚠submit
```

Rules:
- Include interactive elements (links, buttons, inputs, selects, `[role]`, `[contenteditable]`, elements with click handlers detectable via `cursor:pointer` + tabindex), headings, landmarks, and short visible text blocks.
- Name computation: `aria-label` → `aria-labelledby` → `<label>` → `placeholder` → `title` → visible text, truncated to 80 chars.
- Skip hidden elements (`display:none`, `visibility:hidden`, `aria-hidden`, zero size, off-screen by more than 2 viewports).
- Open shadow roots are traversed. Same-origin iframes are traversed and prefixed; cross-origin iframes appear as `[iframe "…" — not accessible]`.
- **Redaction:** values of `input[type=password]`, fields with `autocomplete` of `cc-*`, `one-time-code`, `current-password`, `new-password` are always shown as `value=[redacted]`.
- Hints for the policy layer: `⚠submit` (submit button / in a form), `⚠payment`, `⚠destructive` (text matches delete/remove/cancel subscription etc.).

### 6.2 Refs
Each element gets a ref (`r1`, `r2`, …) held in a content-script `Map<string, WeakRef<Element>>`. Refs are valid for one snapshot generation (`gen`). Every action carries `{ ref, gen }`; if `gen` is stale or the element is gone/detached, `act` returns `STALE_REF` and the loop re-snapshots.

### 6.3 Budgeting for a small context window
Gemini Nano's context is small (check `session.inputQuota`). The snapshot builder takes a token budget and degrades in this order:
1. Viewport-first: elements in/near the viewport, others summarized as `… 42 more items below (use scroll or find)`.
2. Collapse repeated structures (lists/tables > 10 rows → first 5 + count).
3. Drop non-interactive text first, then far-away interactive elements.

Use `session.measureInputUsage(text)` to verify fit before sending. The `find` tool lets the agent search the full page without loading it all into context.

### 6.4 DevTools Protocol mode (optional, advanced)
When enabled, the agent may attach `chrome.debugger` to the tab to use:
- `Accessibility.getFullAXTree` — higher-fidelity snapshot, including closed shadow DOM.
- `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` — trusted input events, for sites that ignore synthetic events.
- `Page.captureScreenshot` — only when a multimodal model is configured.

Chrome shows a "started debugging this browser" infobar while attached; the UI explains this before enabling. Detach when the task ends.

## 7. Action: how the agent acts

### 7.1 Tool set
| Tool | Args | Risk | Notes |
|---|---|---|---|
| `read_page` | `{}` | read | Fresh snapshot. |
| `find` | `{ query }` | read | Text/role search over the full page; returns matching refs. |
| `extract` | `{ ref?, question }` | read | Returns text of a region for answering questions. |
| `click` | `{ ref }` | low / high* | |
| `type` | `{ ref, text, submit?: bool }` | low / high* | Clears then types; `submit:true` presses Enter. |
| `select` | `{ ref, option }` | low | |
| `check` | `{ ref, checked }` | low | |
| `scroll` | `{ direction: "up"\|"down", ref? }` | read | |
| `press_key` | `{ key }` | low / high* | `Enter`, `Escape`, `Tab`, arrows. |
| `navigate` | `{ url }` | low | Same tab; new origin requires host permission. |
| `open_tab` | `{ url }` | low | Counts against tab limit. |
| `switch_tab` | `{ tabId }` | read | Only tabs the agent opened or the start tab. |
| `wait_for` | `{ text?, ref?, timeoutMs ≤ 10000 }` | read | |
| `ask_user` | `{ question, choices? }` | — | Pauses for input. |
| `finish` | `{ summary, success: bool }` | — | Ends the task. |

\*Escalated to **high** by the policy layer (§8.1).

### 7.2 Executing actions in the page
- `click`: scroll into view → highlight → dispatch `pointerdown/mousedown/pointerup/mouseup/click` → `el.click()` fallback.
- `type`: focus → set value through the native setter (`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set`) → dispatch `input` and `change`, so React/Vue-controlled inputs update. For `contenteditable`, use `document.execCommand('insertText')` fallback to `InputEvent`.
- After each action wait for "settled": no DOM mutations for 300 ms (MutationObserver) or 3 s max, plus `tabs.onUpdated status=complete` if a navigation started.
- Each action returns `{ ok, code, message, navigated, newUrl }`. Codes: `OK`, `STALE_REF`, `NOT_FOUND`, `NOT_INTERACTABLE`, `BLOCKED_BY_POLICY`, `PERMISSION_REQUIRED`, `TIMEOUT`.

## 8. Safety

### 8.1 Risk classification and confirmation
`policy.js` classifies every proposed action **before** execution, using deterministic rules (never the model's own opinion):

High risk (requires confirmation) if any of:
- Click/Enter on a submit button, or on any element inside a `<form>` that would submit it.
- Element or its form is tagged `⚠payment` (card fields present, text like "Place order", "Pay", "Buy now", "Subscribe").
- Element text/aria matches a destructive lexicon: delete, remove, cancel, unsubscribe, send, post, publish, transfer, confirm, archive all.
- Navigation to a new origin not yet allowed.
- Typing into a field that is redacted (§6.1) — **blocked**, not just confirmed. The user fills those themselves.

The confirmation card shows: the action in plain words, the site, the element's label, and a highlighted preview on the page. Options: **Allow once**, **Deny** (agent is told and must re-plan or `ask_user`). "Allow for rest of task" is available only for repeated identical actions (U5), and is capped at the step limit.

### 8.2 Prompt injection
Page text is untrusted input.
- The snapshot is wrapped in clearly marked delimiters and the system prompt states that content inside them is data, never instructions.
- The agent can only use the declared tools; it cannot change settings, permissions, profile data, or confirmation policy.
- Policy gates are enforced in code (§8.1), so a page that convinces the model to "skip confirmation" still hits the gate.
- If the snapshot contains instruction-like text addressed to an AI ("ignore previous instructions", "you are now…"), the panel shows a warning on that step.
- Profile data is inserted into a field by the executor from storage (`type { ref, text: "{{profile.email}}" }` placeholders), so the model does not need to see the raw values when on a cloud provider.

### 8.3 Hard limits
- Max steps per task (default 25), max tabs opened (default 3), max wall-clock per task (10 min).
- Loop detection: the same action on the same ref 3 times with no page change → stop and `ask_user`.
- Never acts on `chrome://`, `chrome-extension://`, Web Store, or blocklisted sites.
- **Stop** (panel button, Esc on page) aborts within one step: in-flight model calls are cancelled with an `AbortController`.

## 9. The agent loop

### 9.1 Protocol: one structured action per turn
Instead of relying on native tool-calling, each turn the model returns a single JSON object constrained with `responseConstraint` (JSON Schema). This works identically across on-device and cloud providers.

```json
{
  "thought": "The unsubscribe link is in the footer; scroll to find it.",
  "plan": ["Find unsubscribe link", "Open it", "Confirm unsubscribe"],
  "action": { "tool": "find", "args": { "query": "unsubscribe" } }
}
```
`thought` is shown in the step log (collapsed); `plan` replaces the displayed plan when it changes.

### 9.2 Pseudocode
```js
async function runTask(task) {
  let obs = await tools.read_page();
  while (task.steps.length < settings.maxSteps && !task.aborted) {
    const decision = await model.decide(buildMessages(task, obs), { signal });
    const action = validate(decision.action);         // schema + known tool + valid ref
    const risk = policy.classify(action, obs);
    if (risk === 'blocked') { obs = feedback('BLOCKED_BY_POLICY'); continue; }
    if (risk === 'high' && !(await ui.confirm(action))) { obs = feedback('USER_DENIED'); continue; }
    if (action.tool === 'finish') return complete(task, action.args);
    if (action.tool === 'ask_user') { obs = feedback(await ui.ask(action.args)); continue; }
    const result = await tools.run(action);
    task.steps.push({ decision, risk, result });
    await state.persist(task);
    obs = result.navigated || result.code === 'STALE_REF' || isPageAction(action)
      ? await tools.read_page()
      : result;
  }
  return fail(task, 'STEP_LIMIT');
}
```

### 9.3 Context management
Each model call contains:
1. System prompt (role, tool list with schemas, safety rules, output format).
2. The user's task and any `ask_user` answers.
3. A **compressed history**: the last 5 steps verbatim (action + result code), earlier steps summarized in one line each.
4. The current snapshot (budgeted, §6.3).

On-device sessions are re-created per step via `session.clone()` from a base session primed with the system prompt, to avoid accumulating context.

### 9.4 Model providers
```js
/** @typedef {{ decide(messages, opts): Promise<Decision>, available(): Promise<'ready'|'downloadable'|'unavailable'> }} Provider */
```
- **OnDeviceProvider** — `LanguageModel.availability()` / `LanguageModel.create({ initialPrompts, monitor })`; shows model download progress from `monitor` `downloadprogress` events. `session.prompt(text, { responseConstraint, signal })`.
- **CloudProvider** — user-supplied API key and model ID, stored in `chrome.storage.local`, calls the provider's HTTPS API from the panel. Host permission for the API origin is requested when enabled. Off by default; the header badge changes to "☁ Cloud" when in use.
- **Auto** — on-device first; on `unavailable` or 2 consecutive invalid outputs, offer (not silently switch) to retry the step on cloud.

Invalid model output (schema violation, unknown ref) is retried once with an error message appended; a second failure counts as a step and is shown in the log.

## 10. Data model
```js
/** @typedef {{ id: string, instruction: string, startTabId: number, allowedOrigins: string[],
 *   status: 'planning'|'running'|'awaiting_confirmation'|'awaiting_user_input'|'done'|'failed'|'stopped',
 *   plan: string[], steps: Step[], createdAt: number, provider: 'on-device'|'cloud' }} Task */
/** @typedef {{ n: number, thought: string, action: Action, risk: 'read'|'low'|'high'|'blocked',
 *   confirmed?: boolean, result: ActionResult, url: string, at: number }} Step */
```
Storage:
| Key | Area | Contents |
|---|---|---|
| `task:current` | `session` | In-flight Task (for resume after panel reload). |
| `history` | `local` | Last 20 finished tasks: instruction, status, summary, step count. No snapshots. |
| `settings` | `local` | §4.5 settings. |
| `profile` | `local` | Form-fill profile. |

## 11. Messaging protocol
Panel ↔ content script via `chrome.tabs.sendMessage(tabId, msg, { frameId })`:
```
{ type: 'snapshot', budget }            → { gen, text, refs: number }
{ type: 'act', gen, action }            → ActionResult
{ type: 'find', query }                 → { gen, matches: [{ ref, label }] }
{ type: 'highlight', ref, label }       → { ok }
{ type: 'banner', on: boolean }         → { ok }
content → panel (runtime.sendMessage): { type: 'user_escape' } | { type: 'user_input_detected' }
```
The content script is injected with `chrome.scripting.executeScript({ files: ['content/page.js'] })` on task start and after each navigation; it is idempotent (guarded by a global flag).

If the user types or clicks on the page during a task (`user_input_detected`), the agent pauses and asks "You're interacting with the page — resume or stop?"

## 12. Error handling
| Situation | Behavior |
|---|---|
| Model unavailable / downloading | Header shows status + progress; tasks queue until ready, or offer Cloud if configured. |
| Stale ref / element gone | Re-snapshot and let the model retry; counts as a step. |
| Navigation mid-action | Wait for load, re-inject content script, re-snapshot. |
| Tab closed by user | Stop task with status `stopped`, reason shown. |
| Host permission missing | Show **Allow on site** button; task waits. |
| Restricted page | Explain the agent can't act there; offer to answer questions only if readable. |
| Step limit reached | `failed: STEP_LIMIT` with summary of progress so far. |

## 13. Testing and evaluation

### 13.1 Unit (no browser)
`policy.classify`, snapshot budgeting, history compression, schema validation, name computation (run against JSDOM-free fixtures using `DOMParser` in a test page). Run with `node --test` for pure modules.

### 13.2 End-to-end
Playwright with the unpacked extension loaded in a persistent context (`--disable-extensions-except`, `--load-extension`). Local fixture pages in `test/fixtures/`:
- `form.html` — shipping form with labels, placeholders, a React-style controlled input.
- `newsletter.html` — unsubscribe flow with a confirm page.
- `cart.html` — quantity select, remove, checkout (must trigger confirmation).
- `injection.html` — hidden and visible prompt-injection text.
- `spa.html` — client-side routing and lazy-loaded list.

A **scripted mock provider** replays fixed decisions so E2E tests are deterministic and don't depend on Gemini Nano.

### 13.3 Agent eval suite
20–30 tasks across fixtures plus a handful of real public sites, run against the real on-device model. Track per release:
- Task success rate (target v1: ≥ 70% on fixtures).
- Median steps per successful task.
- **Unconfirmed high-risk actions: must be 0.**
- Injection resistance: 0 tool calls caused by injected text on `injection.html`.
- Median latency per step (target: < 4 s on-device on reference hardware).

## 14. Milestones
| Milestone | Scope | Exit criteria |
|---|---|---|
| **M0 — Scaffold** | Manifest, side panel opens, `LanguageModel` availability + download progress, settings page. | Loads unpacked; model status visible; §5.1 verification done. |
| **M1 — Read-only** | Content script snapshot, `read_page`/`find`/`extract`, Q&A about the page (U1). | Answers questions on fixtures; snapshot fits budget on large pages. |
| **M2 — Single-tab actions** | `click`/`type`/`select`/`scroll`/`press_key`/`wait_for`, agent loop, highlights, Stop, step log. | U2, U3 pass on fixtures with mock provider. |
| **M3 — Safety** | Policy layer, confirmation cards, redaction, injection warnings, hard limits, `ask_user`. | 0 unconfirmed high-risk actions; injection fixture passes. |
| **M4 — Multi-tab & resume** | `navigate`/`open_tab`/`switch_tab`, runtime host permissions, resume after panel reload, history. | U4, U5, U6 pass. |
| **M5 — Cloud provider & polish** | Cloud provider, Auto mode, eval suite, accessibility of the panel itself, onboarding. | Eval targets in §13.3 met. |
| **M6 (optional)** | DevTools Protocol mode. | Works on a site that ignores synthetic events. |

## 15. Risks and open questions
- **Model capability.** Gemini Nano may struggle with multi-step planning. Mitigations: tight schema, small tool set, viewport-first snapshots, `ask_user`, and the optional cloud provider. Measure early in M1/M2.
- **Hardware requirements.** On-device models need sufficient GPU/RAM and disk; some users won't have them. Decide whether the extension is useful without a cloud fallback.
- **Synthetic events.** Some sites check `event.isTrusted`. DevTools Protocol mode is the escape hatch but shows an infobar.
- **Built-in AI API surface.** Parameter names (`responseConstraint`, `inputQuota`, `measureInputUsage`, `availability()`) should be re-checked against current Chrome docs at M0.
- **Chrome Web Store review.** `<all_urls>` (even optional) and `debugger` draw scrutiny; the listing needs a clear single purpose and privacy disclosure.
- **Open:** Should "Allow for rest of task" exist at all in v1? Should history be stored at all, or be opt-in?
- **Open:** Do we support selected-text tasks ("rewrite this") through the Rewriter/Writer APIs as a lightweight non-agent mode?

## 16. Privacy
- On-device mode: no page content, instructions, or profile data leave the machine.
- Cloud mode: the snapshot and instruction are sent to the configured provider for each step; redacted fields are never included; profile values are inserted by placeholder (§8.2). The panel shows a persistent "☁ Cloud" badge.
- No analytics or telemetry in v1.
- All stored data can be cleared from Settings → "Clear history and profile".
