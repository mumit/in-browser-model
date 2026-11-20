# Teaser brief

- Audience: developers on X, day 0 of the "runs in your browser tab" series.
- What it shows: the last 3.4 to 3.8 seconds of each demo (its held result),
  each after a 1 s card with the app's name, between an opening and a closing card.
- Source: each demo's caption-free take (`takes/demo.clean.mp4` and its
  `demo.beats.json`) from the run below, copied into `takes/` and scaled to
  1920x1080. Re-make it after re-recording any demo by copying the new take.
  - debate: demo/takes/runs/2d8c11ad8de54e8580045786e7b4a0e9 (step "room")
  - narrator: demos/nervous-narrator/takes/runs/40c32dc3fae04d82ae43bcc1dc486ba4 ("inspect")
  - outage: demos/outage-adventure/takes/runs/ef4d5351f4b14d12a884a00fba776e2c ("hallway")
  - golf: demos/prompt-golf/takes/runs/ed91e543b6694c3ba9ab2cd169092e06 ("sink")
  - telephone: demos/ai-telephone/takes/runs/f04f512726cb4945a87bef2fa3da3d03 ("flip")
  - slides: demos/follow-slides/takes/runs/e7213f9d995d4fd3bfd2134b0e6505ae ("game")
- Nothing new is recorded; the clips are real time, as recorded.

Make it:

    cd demos/teaser
    reshoot edit edit.toml takes/debate.mp4 takes/narrator.mp4 takes/outage.mp4 takes/golf.mp4 takes/telephone.mp4 takes/slides.mp4
    reshoot check takes/teaser.mp4 takes/teaser-x.mp4 --full --review review
