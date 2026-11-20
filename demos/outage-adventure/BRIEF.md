# Demo brief

- Audience: developers on X, day 3 of the "runs in your browser tab" series.
- Problem they recognize: LLMs make things up, and you can't tell from the prose.
- One task to show: open Outage at 3 A.M. from the gallery, load the command
  matcher and the narrator, type a free-form command, take the flashlight and
  walk into the hallway, with the engine's literal line under each narration.
- Visible result that proves it worked: "rummage through the desk" is matched to
  the real action "open the security desk"; the noir narrator retells each move
  with the engine's literal line underneath; on entering the hallway the
  grounding check flags an item the narrator mentioned that isn't there.
- Why the result matters: the model tells the story and the rules engine keeps
  the facts, all in the browser tab. Nothing leaves the computer.
- Destination and length: X (the `x` export), about 30 seconds.
- What the viewer should do next: open the game and try to make the narrator lie.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so the saved
  game and leaderboard in localStorage start empty: that is the reset.
- Models: MiniLM (~23 MB) for matching commands, Qwen2.5 0.5B as the narrator.
  Loading the narrator switches the style to "Hard-boiled noir detective".
- Not mocked. Loading the models (about 15-20 s) is sped up 4x in the edit and
  labelled "Sped up 4x".
- The narrator samples (temperature 0.5), so its words differ every take. The
  demo doesn't depend on what it says: it shows the narration next to the
  engine's line. The grounding check only knows the game's own vocabulary, so
  most inventions (an alarm clock, a dark alley) pass it; in testing it flagged
  3 of 13 narrations, all for the word "door". The captions don't claim it
  catches inventions.
- Needs Transformers.js 4.3.2 or later in the app: 3.7.6 drifted between
  generations on WebGPU and produced garbled narration.
