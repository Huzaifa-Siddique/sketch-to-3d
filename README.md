# Sketch to 3D — v1 (code-model route)

A website where anyone uploads a photo of a hand-drawn sketch (simple
objects: happy face, kite, cup, pen, pencil, mobile phone) and gets a
rotatable 3D model, then downloads it as GLB, OBJ, or STL.

## How it works

1. The page (`index.html`) takes a sketch photo (file picker or phone
   camera), shrinks it to max 1024 px in the browser, and POSTs it to
   `/api/generate`.
2. `/api/generate` (`api/generate.js`, a Vercel serverless function) sends
   the photo to OpenRouter and asks a **free** vision+code model to write
   Three.js code that builds the object from simple shapes
   (sphere, box, cylinder, cone, torus, tube).
3. The code comes back to the browser and runs inside a **sandboxed
   iframe** (`sandbox="allow-scripts"`, no same-origin). It can only touch
   Three.js inside the frame. It cannot reach the page, cookies, or the
   network.
4. The preview spins (drag, pinch, buttons — same feel as the smiley
   page). Download buttons export the model with three.js exporters,
   all client-side: GLTFExporter (GLB), OBJExporter, STLExporter.

## Models (free only, checked 2026-11-12)

| Role | Model | Why |
| --- | --- | --- |
| Primary | `nvidia/nemotron-3.5-super-vl:free` | NVIDIA open multimodal MoE, 12B active / 120B total, 1M context, 131K max completion. Takes text + images. |
| Fallback | `google/gemma-4-31b-it:free` | Google DeepMind 30.7B dense multimodal, 256K context, 32K max completion, strong instruction following. |

If the primary fails (429, 5xx, timeout, or unusable code), the function
tries the fallback once. It never retries on 401/402 (key/billing).

Free-model limits on OpenRouter (verified 2026-11-12): **20 requests per
minute, 50 per day** on an account that never bought credits; **1,000 per
day** after a one-time $10 credit purchase (credits never expire). Failed
attempts count. The counter resets at midnight UTC. A class demo can burn
50 in one day — that is a v1 limit, not a bug.

## Deploy

```bash
# 1. In the Vercel dashboard: project → Settings → Environment Variables
#    Add:  OPENROUTER_API_KEY = <the key>   (all environments)
#    The key lives ONLY here. Never in code, never in the repo.
# 2. Deploy this folder (Vercel CLI or dashboard import).
#    No build step. index.html is served static; api/generate.js becomes
#    the /api/generate function.
```

No `maxDuration` is set in `vercel.json` on purpose: Vercel Hobby caps
functions at 10 seconds, and a value above the plan limit fails the
deploy. A slow free model can hit the 10-second cap — the page then shows
"the model took too long, try again" in plain words. On Pro (60s) this
mostly goes away.

## Test mode (no key, no API call)

Open the page with `?mock=smiley` or `?mock=cup`. The "Make 3D model"
button uses a built-in code snippet instead of calling the API. This is
for testing the viewer and the downloads only — it says "(demo)" on the
screen.

## What v1 does

- Upload from phone (camera) or computer (file).
- Rotatable preview: drag, pinch, rotate/zoom/reset buttons.
- Downloads: GLB, OBJ, STL (validated files, see BUILD-REPORT-V1.md).
- Plain-word errors. Never a broken page. Never a key in the client.
- Silent fallback to a second free model.

## What v1 does NOT do

- **FBX export.** Three.js has no FBX exporter. GLB/OBJ/STL only.
- **Perfect copies.** A flat sketch has no depth. The model guesses the
  back side. Fine detail (small buttons, thin lines, text) is often lost.
- **Any object.** Simple objects work. Complex or messy sketches come out
  rough. The code model builds with shapes, it does not sculpt.
- **Fast or unlimited.** Each model takes seconds to a minute. Free tier
  is 50 a day (see above).
- **Neural image-to-3D.** v1 uses a code model (the LLM writes Three.js
  code). Real image-to-3D (Tripo/Meshy/Hunyuan3D) is a different step and
  needs paid credits — a possible v2.

## Files

- `index.html` — the whole site (upload, viewer iframe, downloads).
- `api/generate.js` — the OpenRouter proxy. No dependencies.
- `vercel.json` — minimal config (clean URLs only).
- `BUILD-REPORT-V1.md` — what was tested, live vs mock, and what the key
  unlocks.

## Deployment

The site deploys from GitHub. Push to the `main` branch and Vercel
deploys it to https://sketch-to-3d-ochre.vercel.app automatically.
