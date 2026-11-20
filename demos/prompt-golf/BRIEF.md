# Demo brief

- Audience: developers on X, day 4 of the "runs in your browser tab" series.
- Problem they recognize: small models ramble; getting exact output is a craft.
- One task to show: open Prompt Golf from the gallery, load SmolLM2 135M, and
  play hole 1 ("Hello, World", par 20): a bare prompt misses, one extra word sinks it.
- Visible result that proves it worked: "Hello, World!" (13 strokes) makes the
  model ramble ("I'm glad I could be of use to you…"), verdict "Missed";
  "Say Hello, World!" (17 strokes) returns exactly "Hello, World!", verdict
  "In the cup · 17 strokes · Albatross" (3 under par).
- Why the result matters: a real model, greedy and reproducible, in the tab;
  nothing leaves the computer. Invites replies: can you beat 17?
- Destination and length: X (the `x` export), about 25 seconds.
- What the viewer should do next: play the course and post a score.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so scores and
  courses in localStorage start empty: that is the reset. `[[setup]]` closes the
  "How to play" guide that opens on a first visit, then returns to the gallery.
- Model: SmolLM2 135M (the default), WebGPU q4f16, greedy decoding. Both
  prompts gave identical outputs across repeated runs and on Transformers.js
  3.7.6 and 4.3.2.
- Not mocked or sped up.
