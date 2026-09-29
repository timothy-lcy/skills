---
name: jifchat
description: Generate images and videos on the user's JifChat infinite canvas (jif.dev). Use when the user asks to generate, create, or render an image, product shot, poster, short video, or UGC video with JifChat, or says "/jifchat", "/generate image", "/generate video", or "make a similar UGC video". Results land in a JifChat canvas project and spend the user's JifChat credits.
---

# JifChat

JifChat is an AI design canvas. This skill lets you create canvas projects and run image/video generation nodes in them through the JifChat Canvas API, on the user's own account.

## What this skill can and can't do

**Can**
- Create new canvas projects and generate images / videos in them
- Build complete multi-node canvas workflows (text prompts → image generators → video generators)
- Place product photos and reference stills as `imageUpload` nodes
- Put results on the canvas as properly connected, laid-out nodes that show up in the JifChat UI

**Can't (yet)**
- Touch projects shared with the user (owner-only; others return 404)
- Call a template API on a guess. If the user names a template, ask before creating or running anything (see UGC videos)
- Chat with JifChat's assistant, manage billing, or create API keys

**Never** delete a project, or run generation in a project you didn't create in this session, without asking first.

## Project rule — always fresh

**Create a NEW project for every new user request.** Do NOT list the account's projects and do NOT reuse an existing one — the account may hold unrelated work, and each request deserves its own canvas. Only reuse an existing project when the user *explicitly* refers to a previous generation ("make THAT one slower", "add another scene to the drama") — and ask which one if ambiguous.

```bash
# Step 1 of every request: fresh project
curl -s -X POST "$B/api/v1/canvas/projects" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"title":"<short descriptive title>"}'
# → { "_id": "..." } — use this _id as $PID for the whole request
```

## Setup

1. Sign in to JifChat → **Settings → Balance → API Keys** → **Create**. Copy the `jc_...` key — it is shown once.
2. Save it where this skill can read it (either works):
   ```bash
   install -m 600 /dev/null ~/.jifchat_key && printf '%s' 'jc_YOUR_KEY' > ~/.jifchat_key
   ```
   or `export JIFCHAT_API_KEY=jc_YOUR_KEY` in your shell profile.
3. Optional: set `JIFCHAT_BASE_URL` if the key belongs to a different JifChat environment. Default is `https://chat.jif.dev`. A key only works on the environment it was created on.

> OAuth sign-in (authorize in the browser, no key copy-paste) is planned. Until then, setup is the manual key above.

## Calling the API

Start every shell session with:

```bash
B=${JIFCHAT_BASE_URL:-https://chat.jif.dev}
K=${JIFCHAT_API_KEY:-$(cat ~/.jifchat_key 2>/dev/null)}
[ -n "$K" ] || echo "No JifChat key — see Setup"
```

Then send `-H "Authorization: Bearer ***" -H 'Content-Type: application/json'` on every request. **Never print, echo, or log the key.** If the key is missing, stop and walk the user through Setup. Check the key without spending credits: `curl -s "$B/api/v1/canvas/projects" ...` — 200 = OK, 401 = bad key.

## The two paths — pick by job size

**One generation?** → Path 1 (single run call — the API builds the node structure for you).
**Two or more generations, or a pipeline?** → Path 2 (PUT the whole graph, then run).

### Path 1 — single generation

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"img-1","model":"nano-banana-pro","parameters":{"prompt":"a shiba inu on a surfboard, watercolor"},"position":{"x":320,"y":100}}'
```

- **Always send `position`** — never omit it. The API auto-creates the full structure: the `textNode` holding your prompt (placed 320px left of your position) + the edge + the generator. If a text node is already wired, its text is updated instead — never duplicated.
- Response: `{"success":true,"run":{"status":"completed","outputUrl":"https://...","creditCost":150}}`

Video (async — 202 + poll, see *Polling*):
```json
{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"prompt":"...","duration":5,"aspect_ratio":"16:9","resolution":"720p"},"position":{"x":320,"y":100}}
```

To animate a generated image, pass its `outputUrl` as a reference: `"image_urls":["<outputUrl>"]`, cite it in the prompt as `@Image1`.

### Path 2 — multi-node workflow: PUT the whole graph

For 2+ generations (multi-scene videos, storyboard→video pipelines, avatar image + avatar video): **declare the entire node graph in ONE `PUT`**, then run the nodes. Do NOT issue per-node create calls; compose the structure once.

```bash
curl -s -X PUT "$B/api/v1/canvas/projects/$PID" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d @- <<'EOF'
{ ...full graph JSON — see worked example below... }
EOF
```

**⚠ The PUT replaces the ENTIRE nodes/edges arrays — anything you omit is DELETED.** On a fresh project (your default) the arrays are empty, so this is safe. If you must PUT into a project that already has content: GET it first, keep every existing node untouched, add only your new ones, send the complete arrays back.

**Node shape** — every node needs `id`, `type`, `position` `{x,y}`, and type-specific `data`:
- `textNode`: `{"text": "<the prompt>"}`
- `imageGenerator`: `{"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}`
- `videoGenerator`: `{"model": "seedance-2.5", "status": "idle", "resultVideo": null, "errorMessage": null, "prompt": ""}`

**Edge shape** — every edge needs `id` (e.g. `"edge-1"`), `source`, `target`, `sourceHandle`, `targetHandle`, `type: "highlightable"`, `animated: false`, `style: {"stroke": "<color>"}`:
- text: `text-out` → `text-in` (at most ONE text edge per generator), stroke `"var(--handle-text-lit)"`
- images: `image-out` → `image-in-1`, `image-in-2`, … (one edge per slot), stroke `"var(--handle-image-lit)"`

**Layout formula — use it, don't improvise:** each generator occupies a column 450px apart; its textNode sits 320px left of it; rows 240px apart.

**Worked example — 2 image scenes feeding a video:**

```json
{
  "nodes": [
    {"id": "scene1-prompt", "type": "textNode", "position": {"x": 0, "y": 100}, "data": {"text": "scene 1: a dog discovers a surfboard on the beach, watercolor"}},
    {"id": "scene1", "type": "imageGenerator", "position": {"x": 320, "y": 100}, "data": {"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}},
    {"id": "scene2-prompt", "type": "textNode", "position": {"x": 0, "y": 340}, "data": {"text": "scene 2: the dog rides a wave at sunset, watercolor"}},
    {"id": "scene2", "type": "imageGenerator", "position": {"x": 320, "y": 340}, "data": {"model": "nano-banana-pro", "status": "idle", "resultImage": null, "errorMessage": null, "prompt": ""}},
    {"id": "video-prompt", "type": "textNode", "position": {"x": 770, "y": 220}, "data": {"text": "@Image1 discovers the wave, @Image2 rides it — 5s cinematic"}},
    {"id": "vid-1", "type": "videoGenerator", "position": {"x": 1090, "y": 220}, "data": {"model": "seedance-2.5", "status": "idle", "resultVideo": null, "errorMessage": null, "prompt": ""}}
  ],
  "edges": [
    {"id": "edge-1", "source": "scene1-prompt", "sourceHandle": "text-out", "target": "scene1", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-2", "source": "scene2-prompt", "sourceHandle": "text-out", "target": "scene2", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-3", "source": "video-prompt", "sourceHandle": "text-out", "target": "vid-1", "targetHandle": "text-in", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-text-lit)"}},
    {"id": "edge-4", "source": "scene1", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-1", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}},
    {"id": "edge-5", "source": "scene2", "sourceHandle": "image-out", "target": "vid-1", "targetHandle": "image-in-2", "type": "highlightable", "animated": false, "style": {"stroke": "var(--handle-image-lit)"}}
  ]
}
```

The video generator gets its own textNode for the directing prompt; the scene images feed its `image-in-1`/`image-in-2` slots; cite connected images in prompts as `@Image1`, `@Image2` in wiring order.

**Then run the nodes** — same `nodeId` as in the graph, **omit `parameters.prompt`** (the prompt already lives on the canvas; re-sending it would overwrite your text node):

```bash
# images first (sync — result is in the response)
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"nodeId":"scene1","model":"nano-banana-pro","parameters":{}}'

# then the video (async — poll)
curl -s -X POST "$B/api/v1/canvas/projects/$PID/run-video-node" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' \
  -d '{"nodeId":"vid-1","model":"seedance-2.5","parameters":{"duration":5,"aspect_ratio":"16:9","resolution":"720p"}}'
```

Run nodes in dependency order (upstream first); each run spends credits only on success.

## Polling (video only)

After `run-video-node` returns `{"run":{"status":"running","responseUrl":"..."}}`, poll every ~10s:

```bash
curl -s -X POST "$B/api/v1/canvas/projects/$PID/check-run" -H "Authorization: Bearer ***" \
  -H 'Content-Type: application/json' -d '{"responseUrl":"<responseUrl>"}'
```

Until `status` is `completed` (use the output URL) or `failed` (report the error). Tell the user it's in progress — videos take minutes. **Give up after ~8 minutes**: report the run is still going, hand over the canvas link, and note the project persists — the user can ask again later and you'll read the state back instead of re-generating. Never poll past the threshold.

## Models

| Use | Model | Notes |
|---|---|---|
| Image (default) | `nano-banana-pro` | with refs: `${model}/edit` + `image_urls`, ≤14 |
| Image | `gpt-image` | with refs: `${model}/edit` + `image_urls`, ≤16 |
| Video (default) | `seedance-2.5` | refs optional: ≤9 `image_urls`, cite as `@Image1` |
| Video | `grok-i2v` | image-to-video, needs `reference_image_urls` |

Use the defaults unless the user asks otherwise. With reference images (product photo, logo): use `<model>/edit` in the run request only — never store `/edit` in node data.

## UGC videos — one shot, a few times

`examples/kate-canvas.json` is a sanitized export of a finished board (Kate, about 180 nodes). It is **reference only**. Do not clone it, do not PUT it, and do not recreate its nodes. Read it to see how a real shot is wired. The shot to follow is `examples/one-shot.md`.

### Demo 1 — a few shots of one pattern

When the user asks for a UGC video, a commercial in this style, or **"make a similar UGC video"**, build **a few shots** (2–4) of **one** pattern. Each shot is only:

1. **Product upload** — one `imageUpload` of the product.
2. **Reference stills** — `imageUpload` nodes that lock character, place, and framing.
3. **Optional image-generator keyframe** — one `imageGenerator` and its text node, only when those stills must be composed into a single frame first. Skip it when the user already has the frame.
4. **One text node** — the directing prompt for this shot.
5. **One Seedance video node** — `videoGenerator` with model `seedance-2.5`.

Wire text `text-out` → `text-in`. Wire stills `image-out` → `image-in-1`, `image-in-2`, … in the order you cite them. Use `lastframe-in` only for a real end frame. The Kate file keeps the handle ids from that board (including `image-in-0`); when you build, use the handles in this skill.

Layout follows the formula above: one column per shot, the text node 320px left of the video node, rows 240px apart. Product and reference stills sit to the left of that text node.

**Uploads.** If the user handed you files, `POST $B/api/v1/canvas/files/images` as multipart (`file`, plus form fields `width` and `height` in pixels). The response is `{ "url" }`. Put that URL on the `imageUpload` node's `data.imageUrl` and `data.mediaUrl`. If they already gave you an image URL, use that and skip the upload. Uploads are free; generation spends credits. Never echo the API key.

Then run: keyframe images first, then each video node. Omit `parameters.prompt` on the run when the text node already holds the prompt.

### "Make it slower"

"Make it slower", "hold the end", "less shake", and similar notes **edit that video node**. Update its text node (and `duration` when the timing has to change) and re-run the **same** `nodeId`. Do not open a new project. Do not add a shot. Do not touch the rest of the graph.

### Run vs ask

- **"Make a similar UGC video"** (or the same clear request to generate a UGC video in this style) **runs**. Say in one line which shots you are about to run, then run them. Do not wait for a second yes.
- **Template-name jobs** still **ask permission**. If the user names a template — Story to Short Drama, Social Media Post Designs, Before & After, or "use the … template" — stop and ask before creating the project or running anything. A template name is not permission to spend credits.

## Seedance 2.5 prompts

Write the video text node using the [Seedance 2.5 prompt guide](https://docs.volcengine.com/docs/ark/seedance-2-5-prompt-guide?lang=zh). Short rules:

- One continuous shot. The stills define character, product, place, and framing; the prompt defines how they move.
- Open by locking the references in slot order: `@Image1`, `@Image2`, … (the guide writes this as `@图片1`).
- Shape: action, place, one camera move, style, then hard constraints. Say the speed (slowly, suddenly).
- A timeline that fits `duration` (`0–2s`, `2–4s`, …). One main action. Do not stack a locked-off camera and a moving camera.
- End with plain negatives (no extra people, no captions, no cut). Write a spoken line only when the shot needs lip-sync; otherwise say no dialogue.

`examples/one-shot.md` is a real shot written this way.

## Rules

- **Fresh project per request** — see the Project rule.
- **Always send `position`** — Path 1 single node, Path 2 layout formula.
- **Brands:** never generate brand content from text alone — models don't know real logos/colors. Ask for 2–4 reference image URLs (logo, palette, product) and use an `/edit` model.
- **Credits:** each run spends the user's credits, charged on success (image ≈150 on nano-banana-pro). Before a batch (>3 runs) or any video, say what you're about to run and get a yes. An explicit "make a similar UGC video" is already that yes — run it (see UGC videos). A job that names a template is not: ask first.
- **One run per node at a time** — 409 means that node is still running; wait or use a new `nodeId`.
- **Never send an empty array** for an image param (`[]` → 422) — omit the param instead.

## Sync result to canvas (temporary)

Today a successful run does **not** update the node's status on the canvas (it stays `idle` with no image) — structure and positions are handled for you. Until the status write-back lands, after each completed run:

1. `GET $B/api/v1/canvas/projects/$PID` → take `nodes` and `edges`.
2. On the node you ran, set `data.status = "completed"` and `data.resultImage = <outputUrl>` (videos: `data.resultVideo`).
3. `PUT $B/api/v1/canvas/projects/$PID` with `{"nodes": [...], "edges": [...]}` — **all** nodes and edges, unchanged except that one node.

Skip this step if the user has the project open and is editing it — `PUT` replaces the whole node list.

## Errors

| Status | Meaning | Do |
|---|---|---|
| 400 | invalid request (bad model id, bad handle, missing field) | read the error message — it names the problem; fix and retry once |
| 401 | missing/invalid key | re-check Setup and `JIFCHAT_BASE_URL` |
| 402 | out of credits | tell the user to top up in Settings → Balance |
| 404 | project/node missing or not theirs | you probably used a wrong `$PID` — re-check the create-project response |
| 409 | node already running, or edge slot taken | wait, or use a new `nodeId`/slot |
| 429 | rate limited | wait ~30s, retry once |

Full spec: `$B/api/openapi.json` · agent guide: `$B/llms.txt`
