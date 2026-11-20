# Demo brief

- Audience: developers curious about running small language models in the browser.
- Problem they recognize: LLM demos usually need an API key and a server.
- One task to show: open AI Debate Club from the app gallery (index.html), set up a debate in AI Debate Club, load a tiny model in the
  tab, watch two personas argue, and see the judge model's verdict.
- Visible result that proves it worked: two model-written speech bubbles, the
  judge's FOR/AGAINST split with its one-line reason, and the room verdict headline.
- Why the result matters: everything ran locally in a browser tab; nothing left the machine.
- Destination and length: README GIF plus a short captioned MP4, about 25 to 35 seconds.
- What the viewer should do next: open debate-club/debate-club.html and start a debate.

## App and data

- Started by `[app]` in reshoot.toml: a static `python3 -m http.server` on
  127.0.0.1:8777 serving the repository root.
- No account. Each take runs in a fresh headless Chrome profile, so localStorage
  (preferences, league table) starts empty: that is the reset.
- Line-up for speed: SmolLM2 135M for both debaters and the judge, 1 round,
  sound "Silent", "Same every time" ticked (greedy decoding). These are set in
  `[[setup]]` on the Debate Club page (saved in its localStorage), then setup
  returns to the gallery where recording starts.
- Not mocked or sped up: the model really downloads from Hugging Face and runs
  in the tab (WebGPU, q4f16, in headless Chrome), shown in real time. With a
  slow connection the load step takes longer; turn on `trim_waits` in edit.toml
  and `trim_wait` on the load/debate/verdict steps if so (it labels the video
  "Wait shortened").
- The room vote uses the "Room picks FOR" button (no microphone in the take).
