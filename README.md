# Atelier · Leonardo AI Studio Console

A clean, single-file front end for the [Leonardo AI](https://leonardo.ai) API — a designer-facing console to generate, edit, upscale and animate images from one browser tab. No build step, no framework, no dependencies. Just one `index.html` you can drop onto GitHub Pages.

![Atelier — Leonardo console](docs/screenshot.png)
<!-- Replace the line above with a real screenshot once deployed, or delete it. -->

---

## What it is

Atelier wraps Leonardo's REST API in a thoughtful, dark-themed studio interface. The app discovers available models live from Leonardo's `/models` endpoint, builds the right form for each tool, queues your job, polls until it's done, and lays the results out in a results canvas with hover-to-download and open-in-new-tab.

Everything is contained in **a single HTML file** — the markup, the styling, and all the JavaScript. That makes it trivial to host, fork, and audit.

---

## Features

### Create
- **Image Generator** — text-to-image (or reference-guided) generation with full control over model, aspect ratio, image count, negative prompt and seed.
- **Creative Ideation** *(Flow)* — a low-friction "flow state" mode that spreads one line into a wide set of quick options so you can chase a direction instead of a single frame.
- **Background Remover** — strip a background to a transparent PNG using Leonardo's Variations API.

### Editing
- **Image Editor** — instruction-based editing ("make it golden hour, remove the parked car, add fog"). Defaults to an edit-capable Kontext model.
- **Video Editor** *(Advanced)* — reference-guided video-to-video editing from a hosted source clip.

### Video & Upscaling
- **Video Generator** — text-to-motion, plus optional **image-to-video** by uploading a starting frame.
- **Universal Upscaler** — increase resolution with a **Creative** (adds detail) or **Precise** (stays faithful) mode.

### Industry presets
One-click scaffolds that wrap your brief in a tuned prompt + negative-prompt + sensible dimensions, each on a model suited to the job. Every scaffold is editable in the form so you stay in control:

| Preset | Built on | Tuned for |
|---|---|---|
| **Marketing** | Lucid Origin | Campaign-ready commercial visuals with space for copy |
| **Photography** | Lucid Realism | Photoreal lighting and lens cues |
| **Interior Design** | Lucid Realism | Materials, daylight and styling for room visualisation |
| **Graphic Design** | Ideogram v3 | Layouts, marks and type-forward assets |
| **Art** | Phoenix v1 | Painterly, expressive illustration |
| **Architecture** | Lucid Realism | Exterior archviz and material realism |

### Quality-of-life
- **Live model sync** — pulls the current production model list from your account; falls back to a built-in list when offline.
- **Job history** — your recent jobs (capped at 40) persist in the browser via `localStorage`, with a graceful in-memory fallback.
- **Status indicators** — connection and model-sync state shown in the sidebar; per-job pending / ready / failed states.
- **Responsive** — three-column desktop layout collapses to a stacked layout and an off-canvas sidebar on smaller screens.
- **Accessible touches** — keyboard-focusable controls, visible focus rings, and `prefers-reduced-motion` support.

---

## How it works (and how your API key stays safe)

Leonardo's API requires a secret key. **You should never put that key in a public web page** — anyone could read it and spend your credits.

Atelier is built around a small **proxy endpoint** (a [Cloudflare Worker](https://workers.cloudflare.com/)) that holds the key server-side. The browser talks to *your* worker; the worker adds the key and forwards the request to Leonardo. The key never reaches the client.

```
Browser (Atelier)  ──▶  Your Cloudflare Worker  ──▶  Leonardo API
                          (holds the secret key)
```

There is also an **Advanced → direct key** mode in Settings for local testing only (e.g. a CORS-disabled browser or a desktop wrapper). Don't use it on a hosted site, and never commit a key to a public repo.

---

## Setup

### 1. Deploy the proxy worker (~5 minutes)

> If you already have a Leonardo proxy, skip to step 2 and paste its URL.

A minimal Cloudflare Worker that forwards `/v1/*` and `/v2/*` to Leonardo and injects your key:

```js
// worker.js
const LEO = "https://cloud.leonardo.ai/api/rest";

export default {
  async fetch(req, env) {
    const url = new URL(req.url);

    // CORS preflight
    if (req.method === "OPTIONS") {
      return new Response(null, { headers: cors() });
    }

    // (Optional) require an app token so only your site can use the worker
    if (env.APP_TOKEN && req.headers.get("x-app-token") !== env.APP_TOKEN) {
      return json({ error: "Unauthorized" }, 401);
    }

    // Forward the request to Leonardo with the secret key attached
    const target = LEO + url.pathname + url.search;
    const headers = new Headers(req.headers);
    headers.set("authorization", "Bearer " + env.LEONARDO_API_KEY);
    headers.delete("host");

    const res = await fetch(target, {
      method: req.method,
      headers,
      body: ["GET", "HEAD"].includes(req.method) ? undefined : await req.arrayBuffer(),
    });

    const out = new Response(res.body, res);
    Object.entries(cors()).forEach(([k, v]) => out.headers.set(k, v));
    return out;
  },
};

const cors = () => ({
  "access-control-allow-origin": "*", // tighten to your Pages URL in production
  "access-control-allow-methods": "GET,POST,OPTIONS",
  "access-control-allow-headers": "content-type,accept,authorization,x-app-token",
});
const json = (o, s = 200) =>
  new Response(JSON.stringify(o), { status: s, headers: { "content-type": "application/json", ...cors() } });
```

Deploy it and set your secrets:

```bash
npm create cloudflare@latest leonardo-proxy   # choose "Hello World" Worker
cd leonardo-proxy
# paste the worker above into src/index.js

npx wrangler secret put LEONARDO_API_KEY        # paste your Leonardo key
npx wrangler secret put APP_TOKEN               # optional shared token
npx wrangler deploy
```

You'll get a URL like `https://leonardo-proxy.yourname.workers.dev`.

> **Security tip:** in production, change `access-control-allow-origin` from `*` to your exact Pages URL, and set an `APP_TOKEN` so random sites can't burn your credits.

### 2. Point Atelier at your worker

Two options:

**A — paste it in the app.** Open Atelier, click the **gear → Connection**, paste your endpoint URL (and app token if you set one), and **Save & test**.

**B — hard-code a default.** Near the top of the `<script>` in `index.html`:

```js
const DEFAULT_ENDPOINT = "https://leonardo-proxy.yourname.workers.dev";
const DEFAULT_TOKEN    = ""; // must match APP_TOKEN in the worker, if used
```

This way the designer never has to paste anything; the Settings modal still overrides it.

### 3. Host on GitHub Pages

```bash
git add index.html README.md
git commit -m "Atelier — Leonardo console"
git push
```

Then in your repo: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, pick your branch and the root folder, and save. Your console goes live at `https://<user>.github.io/<repo>/`.

---

## Usage

1. **Pick a tool** from the sidebar (Image Generator, Ideation, an Industry preset, etc.).
2. **Fill the form** — write a prompt, choose a model and aspect ratio, set how many images you want. For editing/upscaling/background-removal, drop in a source image.
3. **Run it.** The job appears in the results canvas as "Generating", then fills in when ready.
4. **Hover a result** to **open** it full-size or **download** it.

Need a model that isn't listed? Click **Sync models** in the top bar to refresh the list from your account. Want to pass extra model parameters? Expand **Advanced** in any form and add raw JSON — it's merged into the request.

---

## Project structure

```
.
├── index.html     # the entire app — markup, styles, and logic
└── README.md      # you are here
```

That's it. One file to host, one file to fork.

---

## Tech notes

- **Vanilla** HTML/CSS/JS — no build tooling, no runtime dependencies.
- Fonts: Space Grotesk, IBM Plex Sans, IBM Plex Mono (loaded from Google Fonts).
- Talks to Leonardo's **v1** (init-image upload, variations) and **v2** (generations, models) endpoints.
- Generation is asynchronous: the app creates a job, then polls for completion (up to ~4 minutes) before rendering assets.
- State (endpoint, token, selected section, recent jobs) is stored in `localStorage` with an in-memory fallback so it still runs where storage is blocked.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Network error reaching the endpoint" | Worker not deployed, or CORS blocks your site | Confirm the worker URL works in a browser; allow your Pages origin in CORS |
| "Endpoint not set" | No proxy URL configured | Add it in **Settings**, or set `DEFAULT_ENDPOINT` |
| Models list shows the built-in fallback | Couldn't reach `/v2/models` | Click **Sync models**; check the endpoint and token |
| Upload to storage fails | Storage bucket rejects your origin | Use the proxy (not direct mode) for hosted sites |
| Job stuck on "Generating" then "Timed out" | Slow or failed generation upstream | Retry; video jobs legitimately take longer |

---

## Security checklist

- [ ] API key lives **only** in the worker (as a secret), never in `index.html`.
- [ ] `access-control-allow-origin` restricted to your Pages URL in production.
- [ ] `APP_TOKEN` set if your worker is publicly reachable.
- [ ] No keys committed to the repo's history.

---

## License & attribution

Atelier is an independent front end and is **not affiliated with or endorsed by Leonardo AI**. Use of the Leonardo API is subject to Leonardo's own terms and pricing. Add your preferred license (e.g. MIT) here before publishing.
