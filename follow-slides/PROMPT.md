# Slides That Follow You — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the app from scratch as a single HTML file. The reference build is `follow-slides.html` in this folder.

---

Build **Slides That Follow You**: a single self-contained HTML file for presenting without a clicker. A speech model and a small embedding model run in the browser, listen to the presenter, and advance to the next slide once they're clearly talking about it. Afterwards it gives per-slide talk notes. Audience: AI/ML engineers and PMs. It should also show how embeddings and thresholds work in real time.

## Hard constraints

- **One `.html` file** with inline CSS and JS, opened locally in Chrome or Edge. Not a hosted Claude artifact, because those block model weight downloads.
- **Transformers.js:** load with a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")`.
- **Matcher:** `pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2")`, called with `{pooling:"mean", normalize:true}`. Cosine similarity is then a dot product.
- **Ears**, chosen with a select:
  - Whisper `Xenova/whisper-base.en`, local and private: the default.
  - Chrome `webkitSpeechRecognition`: labelled "faster, sends audio to Google".
- **Persistence:** the deck source and settings in `localStorage` (key `follow-slides-v1`), with every read and write wrapped in try/catch.

## Deck format (Markdown)

- Slides are split on lines containing only `---`.
- `# ` is the title, `## ` a subtitle, `- ` or `* ` bullets, `> ` a quote, and any other line a paragraph.
- `cue: …` lines aren't shown. They're extra phrases for matching.
- Support inline `**bold**` and `` `code` ``. Escape HTML.
- Each slide becomes `{title, sub, items[], cues[], html}`.
- Ship an 8-slide sample deck about running models in the browser: title; why local; what fits; how it runs; Prompt Golf demo; Nervous Narrator demo; the limits; what's next. Each slide has a `cue:` line.

## Matching

- **Index:** each slide's vectors are its whole text (title + items + cues), plus each item and cue as `"Title: item"`. Re-index on Apply deck.
- **Context:** keep the last 3 transcript chunks. Embed the last chunk and the last two joined. A slide's score is the max cosine over all its vectors and both context windows.
- **Eagerness slider** (0–10, default 5), with `e = sens/10`:
  - `margin = 0.14 − 0.11e`
  - `floor = 0.42 − 0.17e`
- **Forward:** only the next slide is considered. When `next ≥ floor` and `next − current ≥ margin`, increment a streak; otherwise reset it. Advance when the streak reaches 2, or right away when the lead is ≥ 2.2 × margin (a "wide margin").
- **Back** (off by default): if `prev ≥ floor + 0.08` and `prev − current ≥ 1.6 × margin` twice in a row, go back.
- **After any move:** clear the context and streaks, so old words don't drag it along.
- **Voice commands** are checked before matching and work even when auto-advance is off:
  - "next slide"
  - "previous slide" / "go back a slide" / "back a slide"
  - "go to / jump to / skip to slide N", with digits or the number words one to twelve
- **Manual control:** arrows, PageUp/PageDown and Prev/Next buttons. F toggles full screen.

## Listening

- **Whisper path:**
  - `getUserMedia` with echo cancellation and noise suppression, into an `AudioContext({sampleRate:16000})`, then a `ScriptProcessorNode(4096)` that collects PCM.
  - Every 3 s of audio, flush a chunk. Skip it if RMS < 0.008 (the HUD shows "quiet").
  - Queue chunks, keeping at most 2 and dropping the oldest so it stays current. Transcribe serially.
  - Strip `[...]` and `(...)` tags, and drop typical silence hallucinations ("you", "thank you", "thanks for watching", "bye", "okay", "so", "uh", "um").
- **Chrome path:** `continuous` and `interimResults` on. Final results feed matching; interim text shows in italics. Restart on `onend` while still listening.
- **Simulate box:** typing a sentence and pressing "Say it" goes through exactly the same pipeline, so you can rehearse without a mic.

## UI

- **Header:** title "Slides That **Follow You**" (second half in the go-green accent), the Ears select, Load models, a progress bar and a status line.
- **Stage:**
  - A 16:9 box using container queries; all slide type is sized in `cqw`.
  - A slide number in the corner.
  - A HUD at the bottom left: a listening dot (red while on), a status text (listening, quiet, transcribing) and a mini bar for how close the next slide is.
  - In full screen, the slide letterboxes to 16:9 and the HUD stays visible.
  - Below the stage: Prev, a counter, Next and Full screen.
- **Cue panel (aside):**
  - **Cue light:** Hold (grey), Standby (amber, within one margin of the bar) or Go (green, "Standby · 1 more" until the streak completes).
  - Start/Stop listening.
  - "Heard" ticker: the last 6 chunks, older ones muted, plus interim text.
  - Simulate input.
  - **Meters** for Previous (only if back is on), Showing, Next and After. Each has a bar and a value. Next has a red tick at the required score `max(floor, current + margin)` and turns green when clear.
  - Auto-advance and Allow-back toggles.
  - Eagerness slider with a text readout (Cautious / Balanced / Eager, plus the actual margin and floor).
  - **"Why it moved" log:** for example `12:03:15 → 4: 0.52 vs 0.31 (wide margin)`, "by hand", or "on command 'next slide'". Back moves are shown in red.
- **Tabs below:**
  - **Deck:** a Markdown textarea, Apply deck and Restore sample deck.
  - **Talk notes:** each transcript chunk is filed under the slide showing when it was heard. Each slide shows its time spent (m:ss, logged on every slide change and stop) and word count. Includes a total, "Copy notes as Markdown" (`## N. Title` + text), and Clear notes with a two-click confirm.
  - **How it works:** pipeline, voice commands, tips for writing a deck that follows well (different vocabulary on neighbouring slides, use `cue:` lines, rehearse with simulate), and privacy.

## Design

- Stage manager's console aesthetic, with cue-light colours.
- **Light tokens:** `--bg #f4f6f8`, `--card #fff`, `--ink #15191d`, `--muted #5f6b76`, `--line #d6dde3`, `--go #178a4c`, `--standby #d89a00`, `--stop #c0392b`, plus stage tokens.
- **Dark override:** `#0d1013` background, `#3cc47c` go.
- **Fonts:** Archivo 800 (display and slides), Inter (body), IBM Plex Mono (data).
- **Layout:** stage plus a 340px aside, stacking below 960px. 16px gutter, no horizontal scroll, visible focus rings, reduced motion respected.

## Verify before handing over

- In Node, test `parseDeck` on the sample (8 titles, cues hidden, quote and bullets rendered) and the command parser.
- Test `decide()` with stubbed scores: a weak lead doesn't move; two clears move; one strong lead moves; back works when enabled.
- Run `node --check` and take a screenshot.
- Tell the user that real listening, model loading and the Chrome speech service can only be tested in their browser.
