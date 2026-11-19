# AI Telephone — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the game from scratch as a single HTML file. The reference build is `ai-telephone.html` in this folder.

---

Build **AI Telephone**: a single self-contained HTML game where a message is passed down a chain of AI "relays", all running in the browser. Each relay speaks the message (TTS), sends it through a simulated phone line (audio degradation), hears it (speech recognition), and retells it in its own words (a small LLM). An embedding model scores how far the meaning has drifted from the original after each relay. Audience: AI/ML engineers and PMs. The lesson is how small errors compound in multi-step pipelines.

## Hard constraints

- **One `.html` file** with inline CSS and JS, opened locally in Chrome or Edge. Not a hosted Claude artifact, because those block model weight downloads.
- **Transformers.js:** load with a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")`.
- **Models**, all loaded by one "Load the line" button with a labelled progress bar (about 650 MB in total):
  - **Matcher:** `pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2")`, called with `{pooling:"mean", normalize:true}`.
  - **Ears:** `pipeline("automatic-speech-recognition", "Xenova/whisper-base.en")`.
  - **Voice:** Kokoro via `import("https://cdn.jsdelivr.net/npm/kokoro-js@1.2.1/dist/kokoro.web.js")`, then `KokoroTTS.from_pretrained("onnx-community/Kokoro-82M-v1.0-ONNX", {dtype:"q8", device:"wasm"})`. `generate(text, {voice, speed})` returns `{audio: Float32Array, sampling_rate}`.
  - **Brain:** `onnx-community/Qwen2.5-0.5B-Instruct` via `AutoTokenizer` + `AutoModelForCausalLM`, with fallback `webgpu/q4f16` → `webgpu/q4` → `wasm/q4` (SmolLM2). Qwen2.5 skips `q4f16`, which outputs garbage on Apple (Metal) GPUs without raising an error: `webgpu/q4` → `wasm/q4`.
- **Persistence:** settings and the leaderboard in `localStorage` (key `ai-telephone-v1`), with every read and write wrapped in try/catch.

## One relay (repeat N times)

1. **Speak:** `kokoro.generate(text, {voice, speed: 1})`. Rotate voices per relay: af_heart, am_michael, bf_emma, am_adam, af_bella, bm_george, af_nicole, am_puck, bf_isabella, am_eric, af_sarah, bm_lewis.
2. **The line:** render through an `OfflineAudioContext(1, len, 16000)`. The source buffer stays at Kokoro's rate; the context resamples it to 16 kHz.
   - For level > 0, run it through a highpass at `280+120·level` Hz and a lowpass at `3600−600·level` Hz.
   - Then add seeded white hiss (amplitude `0.012·level`) and random crackle pops (probability `0.00015·level` per sample, amplitude ±0.35).
   - At level 3, also zero out two dropouts of 1,100 to 2,000 samples each.
   - **Seeding:** mulberry32 seeded with FNV-1a hash of (message + level + style), plus the relay number. The same inputs then always give the same chain, which keeps the leaderboard fair.
3. **Play aloud** (optional toggle, on by default) at 16 kHz in parallel with transcription. Await it before the next relay.
4. **Hear:** Whisper on the degraded audio. Strip `[...]` and `(...)` tags; use "(static)" if the result is empty. Highlight misheard words with a word-level LCS diff between the spoken and heard text (lowercased, punctuation stripped), and count them.
5. **Retell** with greedy decoding (`do_sample: false`, `max_new_tokens: 60`). Clean the output: keep the first non-empty line, strip surrounding quotes, drop a leading "Message:"-style label.
   - **Faithful:** system "You are one person in a game of telephone. Retell the message you heard in your own words as a single sentence. Keep every detail you can. Output only the sentence." The user message is "The message you heard: …".
   - **Gossip:** the same, but "who loves gossip … making it slightly more dramatic".
   - **Via French:** two calls, translating to French and then back to English. Show the French line too.
6. **Score:** cosine similarity between the original and the retelling embeddings, clamped at ≥ 0. The retelling becomes the next relay's input.

"Hang up" cancels the chain and stops the current audio.

## UI

- **Header:** title "AI **Telephone**" (second word in the cable-orange accent), a short description, Load the line, a progress bar and a status line.
- **Left column, the operator's desk:**
  - **The message:** a textarea prefilled with "The deploy is blocked until the auth migration finishes on Thursday." "Hold to say it" records with MediaRecorder and transcribes with Whisper into the textarea. A hint says to aim for 6+ words; under 3 words is refused.
  - **Settings:**
    - Relays (4, 6, 8 default, 10, 12).
    - Line quality: Clear, Office phone (default), Bad cell, Tin can.
    - Retelling style: a Faithful / Gossip / Via French segmented control with help text.
    - Play aloud toggle.
    - Player name for the leaderboard.
  - Start the chain and Hang up buttons.
- **Meaning drift chart (SVG):**
  - Gridlines at 0, .25, .5, .75 and 1, plus a dashed red "lost the plot" line at 0.50, with axis labels.
  - An orange filled area and line through each relay's similarity, starting at 1.0 for relay 0.
  - Points coloured green (≥ .75), amber (≥ .5) or red.
  - Under the chart: **final similarity**, and **relays before it lost the plot** (the count before the first relay under 0.5, or "N (all)").
  - "Copy result" copies a text card: settings, start, end and the drift sequence.
- **Leaderboard:** saved only for completed chains (up to 200 entries, showing the top 10).
  - **Survivors:** highest final similarity.
  - **Chaos:** lowest final similarity, counting only messages of 6+ words.
  - Each row shows the player, "original → final" with the settings as a tooltip, and the score. Clear has a two-click confirm.
- **How it works:** a details section on the pipeline and the compounding-errors lesson.
- **Right column, the line:** a vertical timeline. Each relay is a numbered node on an orange cable (the original is node 0, filled), with a card that shows:
  - Live stage pills: speaking → on the line → listening → retelling → scoring.
  - "Heard" with misheard words highlighted and a count.
  - "In French" for that style.
  - "Passed on" in bold.
  - A coloured similarity badge and the voice used.

  Scroll to each new relay, respecting reduced motion.

## Design

- Telephone exchange aesthetic.
- **Light tokens:** `--bg #edf2f1`, `--card #fff`, `--ink #102624`, `--muted #587370`, `--line #cfdcda`, `--cable #e2611b`, `--good #1e8a5a`, `--warn #c98a00`, `--bad #c0392b`, `--mark #ffe39a`.
- **Dark override:** `#0c1615` background, `#ff8a4c` cable.
- **Fonts:** Unbounded (display), Inter (body), IBM Plex Mono (data).
- **Layout:** a 380px desk plus a fluid line column, stacking below 920px. 16px gutter, no horizontal scroll, visible focus rings, `aria-live` on the line.

## Verify before handing over

- In Node, test `clean()` on quoted and labelled outputs, that the seeded RNG is reproducible, and the LCS diff (e.g. "deploy" → "deep boy" counts as misheard).
- Run `node --check` and take a screenshot.
- Tell the user that model loading, Kokoro, audio playback and per-relay speed can only be tested in their browser, roughly 5 to 8 seconds per relay with WebGPU.
