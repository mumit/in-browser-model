# Outage at 3 A.M. — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the game from scratch as a single HTML file. The reference build is `outage-adventure.html` in this folder.

---

Build **Outage at 3 A.M.**: a single self-contained HTML text adventure for on-call engineers. A deterministic JavaScript engine owns the truth (rooms, items, locks, puzzle state). Two optional models, running in the browser, each do one job: a small embedding model (MiniLM) turns free-form commands into one of the actions that are valid right now, and a small language model rewrites the engine's narration in a chosen style. A grounding check flags anything the narrator invents. The game must be fully playable with no download, using the rules parser and plain narration. Audience: AI/ML engineers and PMs. The lesson is grounding, tool calling, and catching hallucinations.

## Hard constraints

- **One `.html` file** with inline CSS and JS, opened locally or on GitHub Pages. Not a hosted Claude artifact, because those block model weight downloads.
- **Matcher (optional, ~23 MB):** `pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2")` with `{pooling:"mean", normalize:true}`, loaded by its own button.
- **Narrator (optional):** load `onnx-community/Qwen2.5-0.5B-Instruct` (default, ~790 MB) or `onnx-community/Qwen2.5-1.5B-Instruct` (~1.8 GB) with Transformers.js 3.7.6 via a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")`, using `AutoTokenizer` + `AutoModelForCausalLM`.
  - Backend fallback: `webgpu/q4f16` → `webgpu/q4` → `wasm/q4` (SmolLM2). Qwen2.5 skips `q4f16`, which outputs garbage on Apple (Metal) GPUs without raising an error: `webgpu/q4` → `wasm/q4`.
  - Wrap downloads in a retry: on a network error, retry the same file up to 3 times with backoff, and never fall back to another format because of a network error.
- **Voice input (optional):** hold to talk, recording with MediaRecorder and transcribing with `Xenova/whisper-base.en`, loaded lazily once Transformers.js is loaded. **Read aloud** uses the browser's `speechSynthesis`.
- **Persistence:** prefs, the in-progress game state and the leaderboard in `localStorage` (key `outage-adventure-v1`), with every read and write wrapped in try/catch. Reloading resumes an unfinished game with "Welcome back".

## World

**Rooms** (start in the Lobby):

| Room | Description | Exits | Items | Fixtures |
|---|---|---|---|---|
| Lobby | Deserted at 3 a.m., security desk under a flickering light | north → Hallway | coffee mug | security desk (drawer holds the flashlight) |
| Hallway | Hums with air conditioning; a heavy door with a card reader leads north | south → Lobby, east → Kitchen, west → Ops room, north → Server room | — | server room door |
| Kitchen | Still smells of burnt popcorn | west → Hallway | cold pizza | fridge (holds the keycard) |
| Ops room | Dashboards glow red; a laptop is open on the desk | east → Hallway | sticky note ("laptop pw: hunter2") | laptop |
| Server room | Racks roar in the cold air; a breaker panel next to the core switch | south → Hallway | — | breaker panel, core switch |

- **Locked door:** Hallway north needs the keycard. With the keycard in hand, going north auto-unlocks ("You tap the keycard…"). Without it: "The door won't budge… You need a keycard."
- **Dark room:** the Server room is dark unless you carry the flashlight. Without it, describe pitch black with only the doorway south visible, and answer any object command with "too dark".
- **The puzzle:**
  1. Open the drawer and take the flashlight.
  2. Open the fridge and take the keycard.
  3. Take the note, then use the laptop while carrying it, which sets `runbook` ("reset breaker B7, then restart the core switch").
  4. In the Server room, use the panel. It works only with `runbook`; otherwise "Forty identical breakers… You need the runbook." This sets `breaker`.
  5. Use the switch. It works only with `breaker`; otherwise "no power". This wins.
- **The optimal solve is 14 moves.**
- **Flavour:** drink the coffee ("4% more awake"); eat the pizza (consumed, "Morale improves").
- **Items and fixtures** each have a name, synonym words, a description and, for containers, `holds` and `openMsg`.

## Engine (pure functions)

- **State:** `{room, inv[], roomItems{room:[ids]}, opened[], unlocked["room:dir"], visited[], flags{}, moves, hints, won, started, ended}`.
- **`act(state, {action, object, direction})`** handles look (room or object, noting open/closed and changed states), go (by direction or room name), take (including "all"), drop, open (containers move their contents into the room), read, use (the per-item logic above; keycard and door use mean go north), inventory, hint and help. It returns `{text, facts:{room, exits, items, fixtures, event}, changed, room}`.
- **`resolve()`** finds nouns in this order: inventory, visible room items, visible fixtures, then "elsewhere", which produces "There's no X here."
- **Moves:** each command costs 1 move except help. A hint costs 5 moves instead, and gives the first unmet step from an ordered hint list.

## Parsing

1. **Rules parser first.** Lowercase the input and strip politeness ("please", "I want to"). Match regexes for: go, walk, head, enter + a direction or a room name (strip leading "to/into/in/the"); n, s, e, w; take, get, grab, pick up; drop; open, unlock; look; look at, examine, x, inspect, check, search; read; use, flip, press, push, reset, restart, reboot, turn on, log into; eat, drink; i, inventory; hint; help.
2. **If no rule matches and the matcher is loaded, match by meaning.** Build `candidates(state)`: every action that makes sense right now, each with a few natural phrasings:
   - look, inventory and hint;
   - each exit ("go north", "walk to the Kitchen", "leave the Lobby", plus "go back" toward the previous room);
   - take and look for visible items, and open (unopened containers) and look for fixtures;
   - use the laptop, panel and switch when visible; drink coffee and eat pizza; read the note; badge in with the keycard in the Hallway; drop for carried items.
   In the dark room, offer only look, inventory, hint and south. Embed the command and every phrasing (cache the phrase vectors), score each action by its best cosine, and run the top action if it scores at least 0.48. Otherwise reply "Not sure what you mean…" naming the closest action and its score.
   - **Why not a chat model:** tested in Chrome, Qwen2.5 0.5B rarely produced valid JSON and picked from a lettered list little better than chance, while MiniLM matched "raid the fridge", "head back" and "sip of that coffee" correctly and rejected "dance wildly".
3. **Otherwise** say it couldn't understand and suggest commands.

Each command line in the transcript shows a badge: "rules", or "matched “open the fridge” · 0.60".

## Narration and grounding

- **Narrators:** Plain (engine text, no model), Noir detective, Pirate captain, Shakespearean actor, Nature documentary, Sports commentator. Loading the brain switches Plain to Noir.
- **Prompt:** "rewrite one line" framing. The system prompt says: "You rewrite one line of a text adventure in the voice of {style}. Keep exactly the same meaning and facts. Mention only the things in the line. Write one or two short sentences, speaking to the player as \"you\"." Add one example turn per style ("Line: You take the coffee mug." → a styled version), then "Line: {engine text}". Sample at temperature 0.5, top_k 30, repetition penalty 1.15, 60 tokens, at most 2 sentences. Small models still embellish, so always show the engine's line underneath.
- **Grounding check:** build a vocabulary from item and fixture words (over 3 letters, excluding "light"), room first words and the four directions. Flag any vocabulary word in the narration whose canonical name isn't among the current room, exits, visible items or fixtures, inventory, previous room or event text.
  - Show "✓ Grounding check: stuck to the facts." or "⚠ … mentioned X, which isn't here."
  - Show the engine's own sentence under the styled one, as `engine: …`.

## UI

- **Incident banner:** a blinking "SEV-1" chip, the title "Outage at 3 A.M.", "Checkout is down…", and an elapsed timer (starting at the first command) plus a move counter. On a win it turns green with "RESOLVED" and "Fixed in N moves".
- **Terminal card:** a header with traffic-light dots and `oncall@hq: ~/incident`, plus New game (two-click confirm mid-game).
  - The feed shows an intro (PagerDuty page at 3:07 a.m.) and an auto "look". Commands appear in amber with "> ". Room names appear as small uppercase headings.
  - The input row has "Do it" and "Hold to talk". Context-aware quick-command chips below it: look, exits, open or use the visible fixtures, take the visible items, inventory, hint.
  - A win card shows moves, time and hints, and adds the score to the board.
- **Side rail:**
  - **Floor plan (SVG):** visited rooms are filled, the current room is amber, adjacent unknown rooms are dashed "?", and the locked link is a red dashed line until unlocked.
  - **Pockets:** inventory chips.
  - **Models:** two rows, each with its own Load button ("Understand anything you type" for the matcher, "Styled narrator" for the LLM), plus the narrator model select, progress, status, narrator style select and read-aloud toggle.
  - **"Under the hood · last turn":** input, parser, the top three matched actions with scores and the threshold, action JSON, state changed or unchanged with a per-field before → after, engine text, narration and grounding result.
  - **"Fastest fixes" leaderboard:** sorted by moves, then time, noting hints. The empty state says "The best possible is 14 moves."
  - **How it works.**
- **Design:** dark-first 3 a.m. palette: `--bg #0e131f`, `--card #151b2b`, `--card2 #1b2236`, `--ink #e6ebf5`, `--muted #8f99b2`, `--line #273049`, `--sev #ff5a5f`, `--ok #3ddc97`, `--amber #ffb547`, `--link #7fb2ff`. Provide a light override.
- **Fonts:** Archivo (display), Inter (body), IBM Plex Mono (terminal).
- **Layout:** terminal plus a 340px rail, stacking below 960px.

## Verify before handing over

- In Node, run the 14-move solve and assert a win.
- Check failure paths: locked door, wrong room, eat before taking, unknown verb, the hint penalty, and the dark room with "too dark" responses.
- Check that `groundCheck` passes a faithful lobby narration and flags an invented keycard, fridge, door and exit.
- Run `node --check` and take a screenshot.
- Tell the user that model parsing and narration quality need a real browser test.
