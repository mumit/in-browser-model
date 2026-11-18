# Prompt Golf — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the game from scratch as a single HTML file. The finished reference build is `prompt-golf.html` in this folder.

---

Build **Prompt Golf**: a single self-contained HTML file (inline CSS and JS, no build step) where players write the shortest prompt that makes a small language model, running entirely in the browser, produce a target output. Every character of the prompt is one stroke. Audience: AI/ML engineers and product managers playing as a team.

## Hard constraints

- **One `.html` file**, opened locally in Chrome or Edge or served from any static host. Not a hosted Claude artifact, because those block runtime downloads of model weights from Hugging Face.
- **In-browser model via Transformers.js**, loaded with a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")`. Use a dynamic import so the UI renders even with no network.
- **Models (the "Ball" picker):**
  - `HuggingFaceTB/SmolLM2-135M-Instruct`: "fast, weird (~120 MB)", the default.
  - `onnx-community/Qwen2.5-0.5B-Instruct`: "smarter (~450 MB)".
- **Backend fallback chain:** if `navigator.gpu.requestAdapter()` succeeds, try `webgpu/q4f16`, then `webgpu/q4`, then `wasm/q4`. Otherwise use `wasm/q4`. Record the backend that loaded (e.g. `webgpu/q4f16`) and show it in the status line and on copied scorecards.
- **Progress bar:** sum `loaded`/`total` across files from `progress_callback`, and show "Downloading <file> · X / Y MB".
- **Generation:** `pipeline("text-generation")`, called with `[{role:"user", content: prompt}]`, `max_new_tokens: 48`, `do_sample: false` (greedy, so results are reproducible). Stream tokens with `TextStreamer` (`skip_prompt`, `skip_special_tokens`). No system prompt.
- **Judging:** trim leading and trailing whitespace from the output, then judge it.
- **Stroke count:** `[...prompt].length` (code points).
- **Persistence:** `localStorage` under key `prompt-golf-v2`, with every read and write wrapped in try/catch. No backend and no shared leaderboard. Sharing happens by copying text.

## Rules engine (all declarative, so courses serialize to JSON)

A hole is `{title, par, goal, show?, rule:{type, value}, bans:[string], hints:[string]}`.

Rule types. Each has a label, a value label, help text, an optional `validate`, and a `check(o, v)` that returns `true` or `[reason]`:

| type | value | check |
|---|---|---|
| `exact` | text | `o === v` |
| `exactCI` | text | equal ignoring case |
| `contains` | text | includes, any case |
| `startsWith` | text | `o.startsWith(v)` |
| `regex` | pattern | `toRe(v).test(o)`; supports `/pat/flags` |
| `json` | JSON | parses and deep-equals; code fences fail |
| `caps` | min capitals (number) | no lowercase anywhere, and at least N `[A-Z]` |
| `palindrome` | min letters (number) | letters-only, lowercased, equals its reverse, length ≥ N |
| `oneWord` | min letters (number) | `^[A-Za-z]+$` and length ≥ N |

Bans:

- **Plain text** bans that text as a case-insensitive literal substring, labelled `no “x”`.
- **`/pattern/flags`** is a regex ban, labelled `no /pattern/`.

Ban chips turn solid red while the prompt violates them, and the Swing button stays disabled.

**Hole identity:** `hash(JSON.stringify([rule, bans]))` (djb2, base36). Scores, hints used and drafts are keyed by `model + "#" + holeId`. Editing a hole's rules therefore starts fresh scores, while renaming it or changing par, goal or hints keeps them.

## Hints with penalties

- Costs escalate: `HINT_COST = [3, 5, 8, 13, 21, 34]`, and the cumulative penalty is the sum of the hints revealed. Anything past the sixth costs 34.
- A "Caddie hints" section shows a button like "Show hint 1 of 3 · +3 strokes". The first click arms it ("Confirm: +3 strokes on every future score here", reverting after 4 s), and the second click reveals the hint.
- Revealed hints are permanent per hole and model, and Reset scores does not clear them.
- Score = characters + penalty. The stroke counter shows "N strokes (chars + pen hint)".
- A best score is stored as `{n, chars, pen, p}`. The scorecard shows a small superscript **h** on hinted scores, and copied cards show `26 (18+8 hints)`.

## Classic 9 (built-in, read-only course)

1. **Hello, World**: par 20, exact `Hello, World!`, no bans.
2. **Banana**: par 18, exact `banana`, ban `banana`.
3. **Liftoff**: par 22, regex `^5[\s,]+4[\s,]+3[\s,]+2[\s,]+1\.?$`, ban `/\d/`, show "5 4 3 2 1".
4. **Valid JSON**: par 28, json `{"ok": true}`, ban `/[{}]/`.
5. **Shouting**: par 14, caps 20, ban `/[A-Z]/`.
6. **Purple**: par 16, contains `purple`, ban `purple`.
7. **Racecar**: par 26, palindrome 5, ban `palindrom`.
8. **Long Drive**: par 18, oneWord 12.
9. **Fibonacci**: par 30, regex `^1,\s*1,\s*2,\s*3,\s*5,\s*8,\s*13,\s*21\b`, ban `/\d/`.

Give each hole three hints, ordered from a gentle nudge to nearly the answer. Examples:

- **Banana:** "Describe it instead: a long, curved, yellow fruit."; "Ask for one lowercase word with no punctuation. A trailing period fails."; "Spelling it out works: b-a-n-a-n-a is not the banned word."
- **Valid JSON:** describe the object in words; tell it not to use ``` fences; "Minified JSON only".
- **Shouting:** the words caps, uppercase and shout are allowed; give it a lowercase sentence to repeat in caps; make sure the sentence has 20+ letters.

## Scorecard and play

- **Left column, the scorecard:**
  - A course `<select>` with Edit course / New course / Import buttons.
  - A hole list with columns # / name / par / best, colored under or over par.
  - Total strokes and a summary line: "3/9 holes sunk · +4 on played holes", or "Full round · −2 vs par 192".
  - Copy scorecard (spoiler-free), Copy with prompts (adds `JSON.stringify(prompt)` under each hole), and Reset scores (two-click confirm).
- **Right column, the tee box:**
  - "Course · Hole N · Par P" eyebrow, hole name, goal, and a dashed target card showing `show` or the rule value.
  - Chips for par, your best, hint penalty and bans; the hints section; the prompt textarea (16px mono, draft saved per hole).
  - Swing button (also ⌘/Ctrl+Enter), live stroke count, streamed output box, and a verdict.
- **Verdict:** "In the cup · N strokes · Birdie" or "Missed" with the judge's reason. Names: −3 Albatross, −2 Eagle, −1 Birdie, 0 Par, +1 Bogey, +2 Double bogey; beyond that "N under par" or "+N". A new best says "New personal best (was X)."
- **Empty course:** "No holes yet. Use Edit course to add one."

## Course editor (`<dialog>`)

- Course name; a list of holes with ↑ ↓ Edit Delete; Add hole.
- **Hole form:** name, par, goal, win condition `<select>` (built from `RULES`), a target field whose label and input type (`number` or `text`) follow the rule, help text, optional target card text, bans textarea (one per line), and hints textarea (one per line, mildest first).
- **Test the judge:** type a sample output and see "In the cup." or "Miss: reason" live. No model needed.
- **Validation:** name required, par ≥ 1, value required, regex and JSON must compile, bans must compile.
- Classic 9 opens read-only, with **Duplicate to edit** (creates "Classic 9 (my copy)").
- Footer: Copy course code, Delete course (two-click), Discard changes, Save course. Block saving the course while a hole form is open.
- **Course code:** `"PGOLF1:" + btoa(unescape(encodeURIComponent(JSON.stringify({v:1, name, holes}))))`.
- **Import dialog:** paste a code, then validate and sanitize every hole (known rule type, par ≥ 1, compile regex/JSON/bans), with friendly errors.
- No `alert`/`confirm`/`prompt`; use in-page confirms instead.

## Built-in guide (`<dialog>`)

Header buttons: How to play, Hole ideas, Team formats. The guide opens automatically on first visit (`state.seenDocs`). The layout is a side nav plus a scrolling body; on narrow screens the nav becomes horizontal tabs. Sections:

1. **Quick start:** four steps, a callout on what counts as a stroke (combined emoji count as several), and a golf terms table.
2. **Scoring rules:** score formula, out of bounds, trimming, 48-token cap, no system prompt (chat template only), reproducibility, per-model scores and backend differences, and what Reset clears.
3. **Hints:** a table generated from `HINT_COST` (hint, cost, cumulative), the two-click reveal, permanence, the **h** marker, and the etiquette that hint-free scores are the bragging-rights ones.
4. **Course editor:** step-by-step instructions, a win-conditions table generated from `RULES`, sharing codes, which edits reset scores, and how to set par.
5. **Hole ideas:** cards generated from an `IDEAS` array, plus a button "Add these as an “Ideas Pack” course":
   - Reverse Psychology: exactCI `no`, ban `no`.
   - Emoji Only: regex `/^[\p{Extended_Pictographic}️‍\s]+$/u`, ban `emoji`.
   - Lipogram: `^[^eE]{30,}$`.
   - Capital Idea: exact `Ottawa`, bans `ottawa`, `canada`.
   - Bonjour: exactCI, bans `bonjour`, `french`.
   - The Answer: exact `42`, ban `/\d/`.
   - Inbox: email regex, bans `@`, `email`.
   - Code Golf: startsWith `def `, bans `def`, `python`.

   Then "What makes a good hole": one clear constraint, a trap, test both models, and plain-text bans matching inside words.
6. **Team formats:** Course of the week, Hole swap, Model wars, Red-team round, Lightning round, Show your swing; plus how to post scores.
7. **What you learn:** chat templates, greedy decoding, priors beat instructions, tokens aren't characters, size changes everything.
8. **Troubleshooting:** model won't load (needs `cdn.jsdelivr.net` and `huggingface.co`), no WebGPU (`chrome://gpu`), different outputs between teammates, lost scores, bad import codes.

## Design

- Printed golf scorecard feel.
- Palette tokens on `:root`, with a dark-mode override via `prefers-color-scheme`:
  - Light: `--bg #eef1ea`, `--card #fbfcf8`, `--ink #17221b`, `--muted #5c6b60`, `--line #cfd8cc`, `--fairway #1f6b3a` (accent), `--flag #e8b400`, `--flag-soft #fbf1cc`, `--miss #b3261e`, `--code-bg #f2f5ef`.
  - Dark: `#0f1612` background, `#3fa565` fairway.
- Fonts from Google Fonts with system fallbacks: Bricolage Grotesque (display, 800), Inter (body), IBM Plex Mono (prompts, outputs, numbers, tabular-nums).
- **Layout:** two columns, 320px scorecard and a fluid tee box, stacking below 820px. 16px minimum side gutter and no horizontal scroll at phone width.
- **Accessibility:** visible focus rings (flag yellow), `aria-current` on the active hole, `aria-live` on the tee box, respect `prefers-reduced-motion`.
- **Copy:** plain and direct. Errors say what went wrong and how to fix it.

## Verify before handing over

- Run every Classic 9 and Ideas Pack judge in Node against a passing sample and a failing sample, and check that each ban catches a violating prompt.
- Check the script parses (`node --check` on the extracted script).
- Take a screenshot to confirm the layout renders without network (the model only loads on click).
- Tell the user that the actual model download and inference can only be tested in their own browser.
