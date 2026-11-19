# In-Browser Model Lab

Six small apps and games that run AI models entirely in the browser (WebGPU, with a CPU fallback) using [Transformers.js](https://github.com/huggingface/transformers.js) and [Kokoro](https://www.npmjs.com/package/kokoro-js). No server, no API keys. Model weights download from Hugging Face on first use and are cached in the browser.

| App | What it is | Download |
|---|---|---|
| [Prompt Golf](prompt-golf/prompt-golf.html) | Make a tiny LLM hit a target output in the fewest characters, with hints and a course editor | ~120–790 MB |
| [Nervous Narrator](nervous-narrator/nervous-narrator.html) | The model answers out loud, and its voice wobbles where its token probabilities are low | ~880 MB – 1.9 GB |
| [Slides That Follow You](follow-slides/follow-slides.html) | Hands-free slides that advance by matching your speech to slide content | ~105 MB |
| [AI Telephone](ai-telephone/ai-telephone.html) | A sentence passes through AI relays (speak → noisy line → hear → retell), with a meaning-drift chart | ~980 MB |
| [Outage at 3 A.M.](outage-adventure/outage-adventure.html) | An on-call text adventure: a rules engine owns the truth, a small model parses and narrates, and a grounding check flags invented details | 0 MB to play, +23 MB to type anything, +790 MB for the narrator |
| [AI Debate Club](debate-club/debate-club.html) | Two models with silly personas debate out loud; a judge model scores it, with probabilities, and the room votes by applause | ~210 MB – 2 GB |

`index.html` is the landing page. Each folder has the app (a single HTML file) and a `PROMPT.md` that rebuilds it from scratch. `IDEAS.md` lists more ideas.

![AI Debate Club demo: open it from the gallery, two tiny models debate in the browser, a judge model scores it](docs/demo/demo-final-readme.gif)

AI Debate Club with SmolLM2 135M as both debaters and the judge, recorded in real time ([MP4](docs/demo/demo-final.mp4)). The recording source is in `demo/`; re-record it with [reshoot](https://github.com/mumit/reshoot): `reshoot run demo/reshoot.toml`.

## Run locally

Open any HTML file in Chrome or Edge. If the microphone or the model cache don't work from `file://`, serve the folder instead:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

## Publish with GitHub Pages

1. Push this folder to a GitHub repo.
2. Go to **Settings → Pages → Source: Deploy from a branch → `main` / `(root)`**.
3. The site appears at `https://<user>.github.io/<repo>/`.

`.nojekyll` tells GitHub Pages to serve the files as they are, without processing them through Jekyll.

## Notes

- **Browser:** recent Chrome or Edge on a laptop. Without WebGPU, models run on the CPU, single-threaded on GitHub Pages because it can't set cross-origin isolation headers.
- **Networks:** corporate networks may block `huggingface.co` or `cdn.jsdelivr.net`.
- **Storage:** scores, settings and cached models live only in each viewer's browser.
- **Status:** Prompt Golf has been tested end to end in Chrome (download, CPU inference, scoring). The other apps are first builds: model loading, audio and the microphone still need a real-browser test.
- **Downloads:** dropped connections retry the same file automatically (up to 3 times) instead of switching to a bigger model format.

## License

The code in this repo is released under the [MIT License](LICENSE).

The models the apps download (Qwen2.5, SmolLM2, Whisper, MiniLM, Kokoro), and the Transformers.js and kokoro-js libraries loaded from a CDN, are not part of this repo. Each keeps its own license, set by its authors on Hugging Face and npm.
