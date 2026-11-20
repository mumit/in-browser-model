# Demo brief

- Audience: developers on X, day 5 of the "runs in your browser tab" series.
- Problem they recognize: every step of a pipeline is "pretty good", but errors compound.
- One task to show: open AI Telephone from the gallery, load the line, pick
  4 relays on a "Tin can" line with "Gossip" retellings, and run the chain.
- Visible result that proves it worked: "The deploy is blocked until the auth
  migration finishes on Thursday." is misheard as "The deploy is flying…" and
  retold as "…we hear that the deployment has been successfully initiated by the
  team…": the meaning flips. The drift chart falls below "lost the plot"; final
  similarity 0.41.
- Why the result matters: four models (Kokoro voice, Whisper ears, Qwen2.5 0.5B
  brain, MiniLM meaning) chained in a browser tab; nothing leaves the computer.
- Destination and length: X (the `x` export), about 30 seconds.
- What the viewer should do next: say a sentence and see how far it drifts.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so the
  leaderboard in localStorage starts empty: that is the reset.
- "Play each relay out loud" is unticked in `[[setup]]`: the recorder captures
  no sound, and playback only lengthens each relay.
- The chain is reproducible: greedy decoding and line noise seeded from the
  message and settings: the same chain gave the same result in every run and
  fresh session. Needs Transformers.js 4.3.2 or later in the app: on 3.7.6 the
  same chain gave different results within one session.
- Not mocked. Loading the four models is sped up 4x and the 4-relay chain
  (about 40-80 s) 6x, each labelled.
