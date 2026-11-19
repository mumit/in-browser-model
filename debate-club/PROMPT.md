# AI Debate Club — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the game from scratch as a single HTML file. The reference build is `debate-club.html` in this folder.

---

Build **AI Debate Club**: a single self-contained HTML game where two small language models, each playing a ridiculous persona, debate a motion out loud in the browser. A third model judges with probabilities, and the room votes with an applause meter. Audience: AI/ML engineers and PMs. It should be a fun stage moment, and should show persona prompting, sampling, model size, and model-versus-human judging.

## Hard constraints

- **One `.html` file** with inline CSS and JS, opened locally or on GitHub Pages in Chrome or Edge. Not a hosted Claude artifact, because those block model weight downloads.
- **Transformers.js:** load with a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")`.
- **Brains**, any of these per side and for the judge. Load each unique model once into a `Map` (`AutoTokenizer` + `AutoModelForCausalLM`):
  - `HuggingFaceTB/SmolLM2-135M-Instruct` (~120 MB)
  - `HuggingFaceTB/SmolLM2-360M-Instruct` (~275 MB)
  - `onnx-community/Qwen2.5-0.5B-Instruct` (~790 MB)
  - `onnx-community/Qwen2.5-1.5B-Instruct` (~1.8 GB)
- **Defaults:** FOR on Qwen2.5 0.5B, AGAINST on SmolLM2 360M, judge on Qwen2.5 0.5B.
- **Backend fallback:** `webgpu/q4f16` → `webgpu/q4` → `wasm/q4` (SmolLM2). Qwen2.5 skips `q4f16`, which outputs garbage on Apple (Metal) GPUs without raising an error: `webgpu/q4` → `wasm/q4`. Wrap every download in a retry: on a network error (message matching network, fetch, ERR_, connection or timeout), retry the same file up to 3 times with backoff, and never fall back to a different format because of a network error.
- **Voices:** Kokoro via `import("https://cdn.jsdelivr.net/npm/kokoro-js@1.2.1/dist/kokoro.web.js")`, then `KokoroTTS.from_pretrained("onnx-community/Kokoro-82M-v1.0-ONNX", {dtype:"q8", device:"wasm"})`. Fall back to browser `speechSynthesis` (two English voices, pitch 0.9 versus 1.15), or Silent.
- **Persistence:** prefs, the league table and history in `localStorage` (key `debate-club-v1`), with every read and write wrapped in try/catch.
- Show a size hint for the chosen line-up. Changing models after loading re-enables "Load the new line-up".

## Content

- **Ten personas** (id, name, description): Victorian Ghost, Crypto Bro, Pirate Captain, Caffeinated PM, Grumpy Senior Engineer, Sentient Houseplant, Nature Documentary Narrator, Motivational Coach, Medieval Peasant, 1950s Radio Announcer. A custom description field per side overrides the persona; its display name is the text before the first comma or full stop, without a leading article.
- **Seven motions**, all starting "This house…": tabs better than spaces; async standups; hot dog is a sandwich; rewrite it in Rust; ban Friday meetings; pineapple on pizza; dark mode morally superior.
- **Eight Kokoro voices:** am_michael, af_heart, bm_george, bf_emma, am_puck, af_bella, bm_lewis, bf_isabella. Defaults are George for FOR and Puck for AGAINST.

## The debate

- **Rounds:** 1 (openings), 2 (openings and rebuttals, the default) or 3 (adds closing statements). The order is A, B per round.
- **System prompt:** "You are {persona description}. You are in a lively public debate on the motion: "{motion}". You argue FOR/AGAINST the motion. Stay fully in character, be funny and persuasive. Reply with two or three short spoken sentences. No stage directions, no lists, no name labels."
- **User message by turn type:**
  - Opening: "Give your opening statement {stance}." For B, prefix it with A's opening.
  - Rebuttal: "Your opponent, {name}, just said: "…" Rebut them directly and push your case {stance}."
  - Closing: a "short, memorable closing statement".
- **Generation:** `max_new_tokens 80` and `repetition_penalty 1.15`. Sample with temperature 0.8 and top_k 40, unless "Same every time" is ticked (greedy).
- **Tidy the output:** remove `*…*`, `(…)` and `[…]`; strip a leading "Name:" label and surrounding quotes; collapse whitespace; keep at most 3 sentences, trimming each before joining. If empty, use "…I yield my time."
- **Pipelining:** after turn k is written and voiced, start writing and voicing turn k+1 while turn k plays.
- **Playback (Kokoro):** a BufferSource through an AnalyserNode (fftSize 64). Drive a 14-bar VU meter on the speaker's podium and reveal the bubble's words in step with playback time. With the browser voice, reveal on `onboundary`. The speaking podium glows in its side's colour.
- **Stop** cancels the loop and audio.

## The judge

- **Position-bias-corrected probability:** ask the judge model twice with a single-token generation (`max_new_tokens: 1`), with a `logits_processor` hook that reads the first-step logits. Handle Float32, Float16 and Uint16 fp16 arrays. Sum the probabilities of the token ids for "A" and " A", and for "B" and " B", then normalise.
- **Run 1** labels FOR as Debater A; **run 2** swaps the labels. P(FOR) is the average of run 1's P(A) and run 2's P(B). Show both runs and the position bias toward "A" in points.
- **Prompts:** system "You are an impartial debate judge. Judge only on persuasiveness and how well each debater answered the other, not on which side you personally agree with." The user message holds the motion, the labelled transcript, and "Which debater was more persuasive? Answer with a single letter: A or B."
- **Reason:** a one-sentence explanation from a greedy generation (50 tokens), shown in italics.
- **Display:** a FOR/AGAINST split bar.

## The room

- **Applause meter:** per side, 4 seconds of mic capture with echo cancellation, noise suppression and AGC turned off. Measure time-domain RMS each frame, show a live meter and a countdown on the button, and score as the mean of the loudest 30% of frames × 400, capped at 100. It measures loudness only; nothing is recorded.
- Fallback buttons: "Room picks FOR / AGAINST".
- **Result headline:** "{persona} wins the room · the judge agrees / disagrees". A tie asks for another round of applause.
- **League table:** per model, count debates, room wins and judge wins, sorted by room win rate. Recent debates list the motion, personas, models, room winner and judge winner with %. Clear has a two-click confirm.

## UI and design

- **Late-night TV debate set, dark-first:** `--bg #0b0d17`, `--card #141829`, `--card2 #1b2036`, `--ink #eef0fa`, `--muted #9aa0bd`, `--line #262b47`, `--red #ff4d5e` (FOR), `--blue #4d8dff` (AGAINST), `--gold #ffc94d`. Provide a light override under `prefers-color-scheme: light` and `[data-theme]`.
- **Fonts:** Anton (uppercase display), Inter, JetBrains Mono.
- **Header:** "AI **Debate** **Club**" in red and blue, the load button in gold, a progress bar and a status line.
- **Setup card:** a motion input with chips, two side panels (persona, custom description, brain, voice), rounds, judge, sound and greedy options, and Start/Stop with the size hint.
- **Stage:** a soft spotlight gradient. Two podiums (stance, persona name, model and voice, VU bars) with a gold "VS". The feed has round tags and speech bubbles: FOR on the left with a red edge, AGAINST on the right with a blue edge.
- **At rest:** show a hand-written example exchange labelled "Example exchange, written by hand".
- **Lower section:** the verdict card (judge box and room box), then the league table and a "How it works" panel.
- Responsive down to phone width, with visible focus rings and reduced motion respected.

## Verify before handing over

- In Node, test `tidy()` (labels, stage directions and quotes removed, no double spaces, 3-sentence cap) and `turnPlan()`.
- Run `node --check` and take a screenshot.
- Tell the user that model loading, voices, the judge's probabilities and the applause meter need a real browser test.
