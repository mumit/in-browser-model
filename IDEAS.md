# In-browser model app ideas

Apps and games that run a small model entirely in the browser (WebGPU, with a CPU fallback). Nothing is sent to a server. Most combine three pieces:

- **Speech to text:** Whisper via Transformers.js, about 80 to 150 MB.
- **Reasoning and tool calls:** a small instruct LLM. Qwen2.5 at 0.5B to 1.5B is about 0.5 to 1 GB.
- **Speaking back** (optional): Kokoro TTS in the browser, about 80 MB, or the browser's built-in speech synthesis.

Audience: AI/ML engineers, product managers and other technical folks.

## Status

| # | Idea | Category | Status |
|---|---|---|---|
| — | Prompt Golf | Game | Built: `prompt-golf/` |
| 1 | AI Telephone | Game | First build: `ai-telephone/` (untested in a browser) |
| 2 | Jargon Bingo | Game | Idea |
| 3 | Robot by voice | Game | Idea |
| 4 | The Nervous Narrator | Experiment | First build: `nervous-narrator/` (untested in a browser) |
| 5 | Any page, voice-controlled | Experiment | Idea |
| 6 | A rubber duck that interrupts | Experiment | Idea |
| 7 | Slides that follow you | Tool | First build: `follow-slides/` (untested in a browser) |
| 8 | Ask your data out loud | Tool | Idea |
| 9 | Voice sound designer | Tool | Idea |
| — | Voice architecture sketcher | Tool | Idea (earlier pick) |
| — | Standup facilitator | Tool | Idea |
| — | Fool the Decider | Game (decision model) | Parked |
| — | Calibration Duel | Game (decision model) | Parked |

---

## Games your team will play

### 1. AI Telephone

**What it is:** Say a sentence, and it goes around a loop: speech recognition → the LLM paraphrases it → it's spoken aloud → recognized again, for 8 to 10 rounds. You hear "the deploy is blocked on the auth migration" turn into something about migrating birds.

**The hook:** A chart shows how far each round drifts in meaning from the original, measured by embedding similarity. Teams compete to say sentences that survive the longest, or that fall apart the fastest.

**What it teaches:** How errors compound in multi-step agent pipelines. Each step is "pretty good", but the chain isn't.

**Building blocks:** Whisper, a small LLM to paraphrase, TTS, and an embedding model (e.g. MiniLM, about 25 MB).

**Open questions:**
- Should each round use the model's own TTS audio as the next input, or add noise to make it harder?
- A difficulty setting could use a smaller Whisper, a higher paraphrase temperature, or a "translate to French and back" step.

### 2. Jargon Bingo

**What it is:** It listens during a meeting, privately and offline, and marks bingo squares when someone says "circle back", "low-hanging fruit" or "let's take this offline".

**The hook:** It matches by meaning, using embeddings rather than keywords, so "let's discuss that separately" still counts. Everyone gets a card, and the first to finish a row wins.

**What it teaches:** Semantic similarity and thresholds, including false positives that are funny on their own.

**Building blocks:** Whisper running in chunks and an embedding model. It needs no LLM.

**Open questions:**
- Sharing cards across phones needs a shared backend. Simplest option: one screen, with players on teams.
- Always-on transcription drains the battery.

### 3. Robot by voice

**What it is:** Talk a robot through a grid maze ("go forward until the wall, turn left, grab the key"). Fewest spoken words wins.

**The hook:** It's Prompt Golf for agents. You watch plans break when the model misunderstands "until".

**What it teaches:** Planning, tool calling, and the gap between instructions and execution.

**Building blocks:** Whisper, an LLM with tools (`move(n)`, `turn(dir)`, `until_wall()`, `pick_up()`) and a canvas maze.

---

## Experiments engineers will want to dig into

### 4. The Nervous Narrator

**What it is:** The model answers out loud, and its voice tracks how sure it is of each word. Confident words come out steady; uncertain ones come out slower, quieter and slightly wobbly.

**The hook:** Ask it the capital of France, then ask about Canada's 14th prime minister, and you'll hear the difference. A transcript is also shaded by confidence.

**What it teaches:** Uncertainty in language models (token probabilities and entropy), and why "confident-sounding" doesn't mean "correct".

**Building blocks:**
- Whisper for the question (typing also works).
- A small LLM with per-token scores.
- TTS that can change speed and volume per chunk of words.

**Risks:**
- Getting per-token probabilities out of Transformers.js generation needs verifying.
- Per-word voice changes sound choppy, so group words into chunks of similar confidence.

### 5. Any page, voice-controlled

**What it is:** A drop-in script reads the buttons and fields on a web page and turns each into a tool the model can call ("filter to Ontario, sort by date, open the third one").

**The hook:** The page describes its own tools.

**What it teaches:** The core of how browser agents work: turning a UI into a tool list, plus the limits of small models on long tool lists.

**Building blocks:** A DOM scanner that builds tool schemas, an LLM with tool calling, and Whisper.

### 6. A rubber duck that interrupts

**What it is:** Explain your design out loud, and it stays quiet until you contradict yourself ("Two minutes ago you said the cache is per-user, now it's shared") or skip over something ("you said 'then it just syncs'; how?").

**The hook:** It's useful, and safe for talking through internal code because it's local.

**What it teaches:** Long-context reasoning on a running transcript, and when an assistant should stay quiet.

**Building blocks:** Whisper in chunks, an LLM that reviews the transcript periodically, and TTS or a visual nudge.

**Risks:** A small model may miss real contradictions or flag false ones. That needs tuning.

---

## Tools that would get used again

### 7. Slides that follow you

**What it is:** Present without a clicker. It matches what you're saying to the slide content and advances when you get there.

**The hook:** Afterwards it gives you notes of what you actually said on each slide.

**What it teaches:** Embeddings and retrieval in real time, and how to set thresholds so slides don't flip at the wrong moment.

**Building blocks:**
- Whisper in chunks.
- An embedding model over the slide text.
- A small LLM for the after-talk notes (optional).
- A simple deck format, such as Markdown separated by `---`.

**Open questions:**
- Only move forward, never back, unless asked?
- Require a short dwell or confidence margin before advancing?

### 8. Ask your data out loud

**What it is:** Drop in a CSV and ask "Signups by city last week, top five?" The model writes SQL that runs on the laptop (via DuckDB-WASM) and shows you the query.

**The hook:** It can't invent numbers. The answers come from the data, not the model.

**What it teaches:** Grounding, and text-to-SQL with a small model.

**Building blocks:** Whisper, an LLM for SQL, DuckDB-WASM and a chart library.

### 9. Voice sound designer

**What it is:** "Make it warmer, add a slow wobble, more reverb like a cathedral." Vague words become precise audio settings.

**The hook:** It's a fun stage demo of mapping language to continuous controls.

**Building blocks:** Whisper, an LLM with a `set_params` tool, and the Web Audio API.

---

## Earlier ideas

- **Voice architecture sketcher:** talk, and a system diagram builds itself (`add_node`, `connect`, `style`, `rename`), with a panel showing each tool call. "Actually, rename that" tests how well the model tracks state.
- **Standup facilitator:** "next speaker", "two-minute timer", "note a blocker for Priya". It reads back a summary of blockers at the end.
- **Talk to your CSV:** charts from voice. This became #8.
- **Fool the Decider** (parked): write the shortest message that's clearly a refund request to a human but that a decision model (e.g. Laya) scores under 10%.
- **Calibration Duel** (parked): you and a decision model both give probabilities for the same questions, and both are scored with a Brier score.

## General notes

- **Hosting:** Claude-published artifacts block model weight downloads, so these ship as single local HTML files, or on any static host.
- **First load:** about 1 GB for tool-calling models, cached after that. Show a progress bar.
- **Microphone:** works from a local file in Chrome. Use push-to-talk unless the idea needs always-on listening.
- **Reproducibility:** use greedy decoding where scores or comparisons matter.
