# ComfyUI on a free Colab T4

<!-- replace USER/REPO below with your actual GitHub path before pushing -->
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/USER/REPO/blob/main/frizzy_comfyui_colab.ipynb)

A ComfyUI runner notebook for the free Colab tier. Four cells, no toggles for things that
should always be on, and every model downloads onto the VM instead of your laptop.

I run everything locally on 4GB and 12GB cards. This is the other option: when the model is
too big for the card you have, or you just don't want to tie your machine up for three hours,
you borrow a T4 for the session. It is free and it works, but be honest about what it is
before you start.

## What a free T4 actually is

15GB of VRAM on 2018 silicon. Compute capability 7.5, which means no bf16, no fp8, no
FlashAttention 2 and no SageAttention. Big models fit. They just don't fly.

- Prefer GGUF Q4_K_M or Q5_K_S over Q8. Lower Q = less VRAM, less RAM, less compute, lower
  quality.
- The session dies at 12 hours, or after about 90 minutes idle.
- No terminal and no background execution on the free tier, so closing the tab ends the run.
- Models live on the session disk (60 to 100GB free) and are gone when the session ends.
  Drive only holds workflows, settings, outputs and your custom node list, which together
  are well under a gigabyte.

## Quick start

1. Open in Colab, set Runtime to T4 GPU.
2. Add your tokens as Colab secrets (key icon, left sidebar). Optional, see below.
3. Run cell 1, wait for it to finish.
4. Run cell 2. It prints a URL when it has verified that the URL actually works.

Leave cell 2 running. Stopping it stops ComfyUI.

## The four cells

| | Cell | What it does |
|---|---|---|
| 1 | Setup | hardware check, install, Drive mount, restores your last session |
| 2 | Launch | speed layer, a frontend your browser can run, ComfyUI, a watched URL |
| 3 | Save state | pushes workflows, outputs and your node list to Drive |
| 4 | Models + doctor | stage and download models, move ones that landed wrong, diagnose and repair |

There is no workflow cell. Drag a workflow JSON onto the canvas like you would locally.
Whatever it needs comes from the Models panel.

## Models download onto the VM

Clicking a missing model inside ComfyUI normally hands the URL to your browser. On a local
install that is correct, because your browser's download folder IS the ComfyUI machine. Here
it is not, so the file lands on your laptop while the GPU sits empty.

This notebook installs a small custom node that intercepts those clicks and queues them on
the server instead. It also adds a **Models** panel, bottom right.

- **Staging.** Press Download and nothing starts. You get a row per file with the filename,
  the size, the destination, duplicates found elsewhere, and a folder dropdown. Change what
  you want, then press Start.
- **Folder detection.** It reads the repo's own path first, so
  `split_files/text_encoders/umt5.safetensors` goes to `text_encoders`. Then the filename.
  Only then the extension. Anything worked out from the extension alone is flagged amber,
  because `.safetensors` says nothing about what a file is, which is how everything used to
  end up in `checkpoints`.
- **Moving.** Everything already in `models/` is listed newest first with a dropdown per
  file. Moving refreshes ComfyUI's model lists without a restart.
- **aria2 on 16 connections**, resumable, with a magic byte check on arrival so a gated repo
  error page never gets saved as a 4KB `.safetensors`.

The same thing is available in cell 4 as a notebook widget, for when ComfyUI is not running.

## Why the UI is not slow

ComfyUI sends `Cache-Control: no-store` on every `.js` and `.css`. On localhost that costs
nothing. Over a tunnel it means the browser refetches the whole 12.6MB app on every page
load. The asset filenames are content hashed, so the notebook pins them and leaves
`index.html` revalidating. Repeat loads go from about 72 requests to about 7.

The frontend chunks are also gzipped once at launch, which takes about three seconds and
turns 12.6MB into 3.8MB on the wire.

API responses are deliberately left uncompressed. Compressing them looked like free speed
and broke `/api/settings` through localhost.run, which gives you a settings dialog that will
not open while generation keeps working perfectly. Not worth it. The tunnel gzips them
anyway.

## Safari and older browsers

ComfyUI's frontend ships regex lookbehind and class static blocks (Safari 16.4+) and the
regex `v` flag (Safari 17+). Those are compile errors, not runtime errors, so the module
graph dies and you sit on the splash screen forever with a perfectly healthy server.

Cell 2 asks your browser directly what it can compile, because Colab's output runs in that
browser. If something is missing, the frontend is rewritten with esbuild for the right target
and served from there. A modern browser skips the rewrite. Nothing to configure.

If you open the URL in a different browser than the one you have the notebook in, force it:

```python
os.environ["FRIZZY_COMPAT"] = "safari15"
```

## The tunnel gets watched

Quick tunnels do not last. pinggy stops at 60 minutes, cloudflared throttles per IP, and both
ssh transports drop when the operator feels like it. From the browser that looks exactly like
ComfyUI dying, when the server is actually untouched and only the URL is gone.

Cell 2 pings through the tunnel every 45 seconds, which also stops an idle one being reaped.
After two misses it rebuilds, tries the transport that was working first, verifies HTTP and
websockets, and prints the new address in the log you are already watching. The current URL
is also written to `/content/tunnel_url.txt`.

| Transport | Websockets | Needs |
|---|---|---|
| cloudflared | yes | nothing, throttled per IP |
| pinggy | yes | nothing, 60 minutes per tunnel |
| localhost.run | yes | nothing |
| ngrok | yes | free authtoken as `NGROK_TOKEN` |
| Colab proxy | no | cannot run ComfyUI, off by default |

Websockets are not optional. ComfyUI blocks on `/ws` while booting, and progress, previews
and finished images all arrive over it. Google's port proxy passes HTTP and drops websockets,
which is why it is disabled here.

## Secrets

Colab secrets, key icon in the left sidebar. All optional.

| Secret | For |
|---|---|
| `HF_TOKEN` | gated HuggingFace repos, passed through to the in-UI downloader |
| `CIVITAI_TOKEN` | CivitAI downloads |
| `NGROK_TOKEN` | the only tunnel that does not depend on Colab's shared IP |
| `GITHUB_TOKEN` | private repos, and 5000 API calls an hour instead of 60 |

## Cell 4, the doctor

Run it when something is wrong. It checks hardware, the install, the frontend package against
the version your ComfyUI revision pins, ComfyUI-Manager's age, the speed layer, every model
file's first eight bytes, Drive, the running server's boot endpoints, whether `/api/settings`
and `/api/object_info` come back as parseable JSON both locally and through the tunnel, and
the log.

With `AUTO_FIX` on it repairs what it safely can: a frontend version mismatch, an outdated
Manager, a stale extension, a symlinked `user/`, error pages saved as `.safetensors`, a port
held by a dead process. Then it prints one block you can paste into an issue.

## When it breaks

| Symptom | Real cause |
|---|---|
| `CUDA out of memory` | VRAM. Drop a quant, or set `lowvram` |
| Process `Killed`, no traceback | host RAM, not VRAM. Keep `--cache-none` on |
| `error while deserializing header` | the download was an error page. Cell 4 finds and deletes those |
| `Torch not compiled with CUDA` | a pip line replaced torch. Restart the runtime, rerun cell 1 |
| Logo forever, server healthy | frontend the browser cannot compile, or a dead websocket |
| Generation runs, UI shows nothing | websocket, or a frontend and backend version mismatch |
| Settings dialog will not open | `/api/*` arriving unparseable, or an outdated Manager |
| Model downloaded to your laptop | the Models panel is not loaded. Rerun cell 2 |
| Everything landed in `checkpoints` | older builds. Move them in cell 4 or the Models panel |
| Tunnel URL dead | the watchdog prints the replacement, also in `/content/tunnel_url.txt` |
| Node missing after a restart | fresh session. Cell 1 restores from your snapshot |

If a page will not load at all, open `/check.html` on the same URL. It runs the capability,
asset, API and websocket checks inside the browser and prints the result on screen, so you do
not need developer tools.

## What is in the repo

```
frizzy_comfyui_colab.ipynb   the notebook, everything is in here
README.md                    this file
```

The custom node the notebook installs (`ComfyUI-FrizzyDownloader`) is embedded in cell 2 and
written to `custom_nodes/` at launch, so there is nothing else to clone.

## Credits

ComfyUI by comfyanonymous. ComfyUI-Manager by ltdrdata. GGUF support via ComfyUI-GGUF, and
most of the GGUF conversions I use come from city96. Tunnels by cloudflare, pinggy,
localhost.run and ngrok. Downloads by aria2.

## Licence

MIT for the notebook. The models you download have their own licences and some of them are
not permissive: MiniMax H3's Community Licence, for one, excludes local deployment in the EU,
UK, US and South Korea without written authorisation. Wan, LTX, Flux, Qwen and Z-Image carry
no such territory clause. Check before you publish what you make.

---

Built by [the_frizzy1](https://www.youtube.com/@the_frizzy1), where I test this stuff on a
real 4GB laptop and say what actually happened.
