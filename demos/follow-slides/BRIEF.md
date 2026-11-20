# Demo brief

- Audience: developers on X, day 6 (the finale) of the "runs in your browser tab" series.
- Problem they recognize: clickers, and slides that never keep up with the talk.
- One task to show: open Slides That Follow You from the gallery, load Whisper
  and MiniLM, and present the built-in deck ("Models that run in your browser")
  by saying four sentences; each moves the deck one slide by meaning.
- Visible result that proves it worked: each sentence advances one slide
  (2 → 3 → 4 → 5) with a "wide margin" logged in "Why it moved", ending on the
  "Demo: Prompt Golf" slide after "Let me show you a little game we built."
- Why the result matters: speech-following slides, matched by meaning on the
  laptop; nothing leaves the computer.
- Destination and length: X (the `x` export), about 30 seconds.
- What the viewer should do next: paste in their own deck and talk.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so the deck,
  notes and settings in localStorage start at the built-in sample: that is the reset.
- Speech is simulated: the sentences are typed into the app's own "Simulate:
  type what you'd say" box, because the headless recorder has no microphone.
  The first talking caption says so. Matching is the same either way (MiniLM
  embeddings); Whisper loads but isn't exercised.
- Not mocked or sped up. Typing runs at 28 characters a second.
- No language model, so the Transformers.js 3.7.6 drift bug doesn't apply.
