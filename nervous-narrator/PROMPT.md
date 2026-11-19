# Nervous Narrator — rebuild prompt

Paste everything below the line into Claude (or another capable coding model) to recreate the app from scratch as a single HTML file. The reference build is `nervous-narrator.html` in this folder.

---

Build **Nervous Narrator**: a single self-contained HTML file where you ask a question (typed or spoken) and a small language model running in the browser answers out loud. Its voice changes with its own confidence: words it was sure of sound steady, and doubtful words come out slower, quieter, after a pause and with a wobble. The answer is shown with each word shaded by confidence, and you can click a word to see what else the model considered. Audience: AI/ML engineers and PMs. The point is to make model uncertainty audible.

## Hard constraints

- **One `.html` file** with inline CSS and JS, opened locally in Chrome or Edge. Not a hosted Claude artifact, because those block model weight downloads.
- **Transformers.js:** load with a dynamic `import("https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.6")` so the UI renders without network.
- **Kokoro TTS:** load with `import("https://cdn.jsdelivr.net/npm/kokoro-js@1.2.1/dist/kokoro.web.js")`, then `KokoroTTS.from_pretrained("onnx-community/Kokoro-82M-v1.0-ONNX", {dtype:"q8", device:"wasm"})`, using voice `af_heart`.
- **Speech recognition:** `pipeline("automatic-speech-recognition", "Xenova/whisper-base.en")`, loaded lazily on the first mic use.
- **LLM picker ("Brain"):**
  - `onnx-community/Qwen2.5-0.5B-Instruct` (default, ~450 MB).
  - `onnx-community/Qwen2.5-1.5B-Instruct` (~1.8 GB).
- **Model loading:** `AutoTokenizer` + `AutoModelForCausalLM`. Backend fallback: `webgpu/q4f16` → `webgpu/q4` → `wasm/q4` (SmolLM2). Qwen2.5 skips `q4f16`, which outputs garbage on Apple (Metal) GPUs without raising an error: `webgpu/q4` → `wasm/q4` (wasm only if `navigator.gpu.requestAdapter()` fails).
- **Voice picker:**
  - Kokoro (wobbles).
  - Browser `speechSynthesis` (instant, no wobble; prefer a local English voice).
  - Silent.

  If Kokoro fails to load, fall back to the browser voice and show a toast.
- **Progress:** a bar that sums `loaded`/`total` across files, labelled Brain, Voice and Ears.
- **Persistence:** prefs and the calibration tally in `localStorage` (key `nervous-narrator-v1`), with every read and write wrapped in try/catch.

## Capturing confidence (the key technique)

Transformers.js `generate()` doesn't return scores, but it accepts `logits_processor: [fn]`. The function is called once per step with `(input_ids, logits)` after the built-in processors, and must return `logits` unchanged. In it:

- Read the last row: `V = logits.dims.at(-1)`, `off = data.length - V`.
- Logits may be `Float32Array`, `Float16Array` or `Uint16Array` (fp16 bits). Write a `half()` converter for the `Uint16Array` case.
- **Pass 1:** find the max and track the top 4 `[id, logit]`.
- **Pass 2:** skipping `-Infinity` and NaN, compute `sum = Σ e^(v−max)` and `se = Σ e^(v−max)·(v−max)`.
- Store `{ top: [[id, prob] × 4], H }` per step, where `prob = e^(v−max)/sum` and entropy `H = ln(sum) − se/sum` (nats).

Generation details:

- **Messages:** system "Answer the question in one or two short sentences. Plain text only, no lists or markdown." plus the user's question.
- **Template:** `apply_chat_template(..., {add_generation_prompt:true, return_dict:true})`.
- **Settings:** `max_new_tokens: 72`, `do_sample: false`.
- **Streaming:** use `TextStreamer` (`skip_prompt`, `skip_special_tokens`) to show text as it's generated.
- **Generated ids:** `Array.from(seq.data, Number).slice(promptLength)`.

## Tokens to words

- **Robust token text:** for each generated token k, decode the ids `0..k` cumulatively with `skip_special_tokens` and take the new suffix as that token's piece. Skip empty pieces (special tokens).
- **Word boundaries:** a new word starts when a piece begins with whitespace and contains non-space text, or when the previous word ends in whitespace. Punctuation attaches to the previous word.
- **Confidence measure** (toggle):
  - **Top choice:** `top[0].prob`.
  - **Spread:** `1/e^H`.
- **Word confidence:** the minimum over its tokens. Remember which token was the shakiest.
- **Bands:** ≥0.85 steady, ≥0.6 mild, ≥0.35 shaky, below that very shaky.
- **Overall confidence:** the geometric mean of token confidences.

## UI

- **Header:** title "Nervous *Narrator*". "Narrator" is in the accent red with a subtle periodic tremble animation, disabled under reduced motion. Includes the Brain and Voice selects, Load models, a progress bar and a status line showing model and backend.
- **Ask card:**
  - A question textarea (Enter asks, Shift+Enter is a newline).
  - Ask, "Hold to talk" (pointer down/up), Stop voice and Replay buttons.
  - Holding the Space bar talks too, when focus isn't in a field.
  - Seven preset chips: capital of France; Canada's 14th prime minister; 17 times 23; 1987 Stanley Cup winner; HTTP 418; who painted the Mona Lisa; first iPhone year.
- **Answer card:**
  - **Polygraph trace (canvas):** a 16px grid, dashed red lines at 0.85 and 0.35, a jittery ink line through each word's confidence (jitter grows as confidence drops), and dots, red for shaky words. A highlight band and a red needle mark the chunk being spoken. Redraw on resize and on theme change.
  - **Answer text:** large display type. Each word is a focusable span with opacity tied to confidence. Mild words get a faint red wavy underline, shaky ones a red tint and wavy underline, very shaky ones red italic with a thick wavy underline. The word being spoken gets a highlighter background.
  - **Stats:** overall confidence, shakiest word with its %, and the token count.
  - **Word detail:** clicking a word shows the top 4 candidate tokens at its shakiest step, with probability bars (chosen token marked ▸), the remaining probability mass, and the entropy.
  - **"Was it right?"** with Correct / Wrong buttons.
- **Side column:**
  - Nervousness slider (0–2×, default 1).
  - "Add 'um' before very shaky words" checkbox. Label it clearly as added for effect, not said by the model.
  - Top choice / Spread toggle with help text.
  - **Calibration check:** average overall confidence when marked right versus when marked wrong, plus a count and a clear button.
  - "How it works" details.

## Voice

- **Chunking:** group consecutive words into chunks of the same band. Break at sentence ends and after 8 words.
- **Doubt:** `u = min(1.5, (1−conf)·nervousness)`.
- **Kokoro path:** pre-generate every chunk first (status "Composing the voice…") with `speed = max(0.6, 1−0.3u)`. Prepend "um," / "uh," / "hmm," to very shaky chunks when fillers are on. Then play the chunks in order:
  - Wait `140·u` ms before chunks where u > 0.15.
  - Play through Web Audio: a BufferSource into a Gain (`max(0.35, 1−0.4u)`).
  - When u > 0.05, add vibrato (an oscillator at 5.5 Hz, depth `55·u` cents into `source.detune`) and tremolo (an oscillator at 7 Hz, depth `0.22·min(1,u)` into the gain).
- **Browser path:** one `SpeechSynthesisUtterance` per chunk, awaiting `onend`, with `rate = max(0.55, 1−0.35u)`, `pitch = min(2, 1+0.3u)` and `volume = max(0.35, 1−0.45u)`. Use a `160·u` ms pause first.
- **Sync:** highlight the chunk's words and redraw the trace for each chunk.
- **Stop:** cancels pending chunks, stops the current source and calls `speechSynthesis.cancel()`.

## Speech input

- Record with `MediaRecorder` on hold, and decode with `new AudioContext({sampleRate:16000}).decodeAudioData`.
- Pass `getChannelData(0)` to Whisper, then ask the transcribed question.
- Ignore results that are only bracketed tags like `[BLANK_AUDIO]`.
- If the mic is blocked, show a toast suggesting typing instead.

## Design

- Polygraph readout aesthetic.
- **Light tokens:** `--paper #f3f5f7`, `--card #fff`, `--grid #dbe2e8`, `--ink #14202b`, `--muted #5d6b78`, `--line #d3dbe2`, `--needle #c4302b`, `--steady #1d5c8c`, `--mark #fff1a8`.
- **Dark override:** `#0e1419` paper, `#ff6b5e` needle, `#6fb3e8` steady.
- **Fonts:** Familjen Grotesk (display), Inter (body), JetBrains Mono (data), with fallbacks.
- **Layout:** main column plus a 300px aside, stacking below 900px. 16px minimum gutter, no horizontal scroll, visible focus rings, `aria-live` on the answer.

## Verify before handing over

- In Node, test `half()` against known fp16 values, `rowStats` on Float32 and Uint16 rows (they must agree), and the word grouping with a fake tokenizer.
- Run `node --check` on the script and take a screenshot.
- Tell the user that model loading, Kokoro and the mic can only be tested in their browser.
