# Demo brief

- Audience: developers on X, day 2 of the "runs in your browser tab" series.
- Problem they recognize: LLMs sound equally sure whether they're right or wrong.
- One task to show: open Nervous Narrator from the gallery, ask an easy question,
  then a harder one, and click the shakiest word to see what else the model considered.
- Visible result that proves it worked: "Paris" comes back with every word steady
  (99%); asked for Canada's 14th prime minister, it answers "Justin Trudeau"
  (it was Lester B. Pearson), and "Justin" is its shakiest word (about 9%,
  wavy-underlined). Clicking "Justin" shows the model's top alternatives.
- Why the result matters: the model's own probabilities, read live in the tab,
  show where it's guessing. Nothing leaves the computer.
- Destination and length: X (the `x` export), about 30 seconds.
- What the viewer should do next: open the app and hear the voice shake.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so localStorage
  (preferences, calibration tally) starts empty: that is the reset.
- Brain: Qwen2.5 0.5B (the default), greedy decoding (the app always decodes
  greedily). Voice is set to "Silent (text only)" in `[[setup]]`, because the
  headless recorder captures no sound; the end card says the voice is in the app.
- Not mocked. The model download and load (about 15 s) is sped up 4x in the edit
  and labelled "Sped up 4x".
- Needs Transformers.js 4.3.2 or later in the app: 3.7.6 drifted between answers
  on WebGPU, which produced a different (garbled) answer for the same question.
