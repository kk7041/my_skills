---
name: aicodewith-image-generation
description: Mandatory skill for ALL image-generation related requests. Always use this skill whenever the user asks to 生图, 生成图片, 画图, 做图, 出图, 海报, logo, icon, illustration, poster, product shot, edit an image, image-to-image, reference image editing, or any task whose deliverable is a generated/edited image. Use AICodeWith image generation API to create images, edit images from reference URLs, poll async image tasks, and return generated image URLs. Do not use generic image generation tools or answer with only prompts unless this skill/API is unavailable or the user explicitly asks only for a prompt.
---

# AICodeWith Image Generation

This is the mandatory workflow for any image-generation or image-editing task. If the user asks for 生图/生成图片/画图/做图/出图/海报/logo/icon/illustration/poster/product shot/image editing, load and follow this skill before using any image tool. Prefer the AICodeWith API path below; only fall back to a generic prompt or another image provider if this API is unavailable, lacks credentials, or the user explicitly requests a different provider/prompt-only output.

Use this skill whenever the user wants image generation through `https://api.aicodewith.com`, including text-to-image and image-to-image with reference URLs.

## Credentials

- API key is stored in environment variable `AICODEWITH_API_KEY`.
- Do not print, reveal, or write the raw key in responses, logs, or generated files.
- If `AICODEWITH_API_KEY` is missing from the subprocess environment, check the active Hermes profile `.env` file (normally `/home/hermes/.hermes/.env`) and load only the value needed for the request; never echo the value. If it is missing there too, tell the user it needs to be configured before using this skill.

## API summary

Base URL: `https://api.aicodewith.com`

Create task:

```http
POST /v1/images/generations
Authorization: Bearer $AICODEWITH_API_KEY
Content-Type: application/json
```

Query task:

```http
GET /v1/tasks/{task_id}
Authorization: Bearer $AICODEWITH_API_KEY
```

Completion result images are in `result_data[].url`.

## Supported request fields

- `model`: required. Prefer `gpt-image-2` unless user explicitly asks for `gpt-image-2-beta`.
- `prompt`: required, non-empty.
- `size`: optional. Recommended values: `auto`, `1:1`, `2:1`, `1:2`, `3:1`, `1:3`, `3:2`, `2:3`, `4:3`, `3:4`, `5:4`, `4:5`, `16:9`, `9:16`, `21:9`, `9:21`.
- `resolution`: optional. `1K`, `2K`, `4K`; only for `gpt-image-2` when `size` is a ratio.
- `n`: optional. `1-10`; only `gpt-image-2` supports multiple images. `gpt-image-2-beta` must use `n=1`.
- `quality`: optional. `low`, `medium`, `high`; only for `gpt-image-2`.
- `image_urls`: optional list of public reference image URLs. Use for image-to-image/reference editing/outpainting.
- `background`: optional; use `auto` by default.

## Interaction rules — mandatory confirmation gate

Before creating any image-generation task, the assistant MUST get explicit user confirmation for these fields, even if the user's message already implies some of them:

- 模型 / `model`
- 比例 / `size`
- 数量 / `n`
- 质量 / `quality`
- 分辨率 / `resolution`

The assistant may propose values, but must not start the API request until the user confirms the full option set.

Required confirmation format:

```text
请确认本次生图参数：
- 模型：...
- 比例：...
- 数量：...
- 质量：...
- 分辨率：...

确认后我再开始生成。
```

Model-specific rules:

1. Always ask/confirm the model first or include it in the parameter confirmation:
   - `gpt-image-2`
   - `gpt-image-2-beta`
2. If using `gpt-image-2`, the confirmation must include actual selectable values for:
   - quality: `low`, `medium`, `high`
   - size/aspect ratio: `auto`, `1:1`, `16:9`, `9:16`, etc.
   - resolution: `1K`, `2K`, `4K` when size is a ratio; use `不适用（auto 比例）` when size is `auto`
   - image count `n`: `1-10`
3. If using `gpt-image-2-beta`, the confirmation must still show all five fields, but mark unsupported fields as fixed/not applicable:
   - 模型：`gpt-image-2-beta`
   - 比例：user-selected or proposed size
   - 数量：`1（beta 固定）`
   - 质量：`不适用（beta 不支持）`
   - 分辨率：`不适用（beta 不支持）`
4. If the user's prompt is missing or too vague, ask for the image prompt before or together with the parameter confirmation.
5. Do not silently apply defaults. Even when the user says “默认 / 随便 / 你决定”, propose the default values in the five-field confirmation and wait for the user to confirm.
6. Only proceed to API creation after the user clearly confirms, e.g. “确认”, “可以”, “就按这个”, “开始生成”.

## Defaults

Defaults are suggestions only. They are never permission to skip confirmation.

- `model`: `gpt-image-2`
- `size`: `1:1`
- `resolution`: `1K`
- `n`: `1`
- `quality`: `medium`
- `background`: `auto`

For quick/cheap drafts, suggest `quality: low`.
For high quality/final artwork, suggest `quality: high`.

## Workflow

1. Parse the user's generation request into API parameters.
1b. **Reference image analysis** — when the user provides a reference image (local file, URL, or Telegram image cache path) and asks to create a derivative visual asset, analyze the reference FIRST before proceeding to parameter confirmation:
   - Extract dominant colors, color distribution, and palette
   - Assess composition: symmetry, visual weight distribution (top/bottom/left/right density)
   - Identify edge density patterns and shape distribution
   - Summarize findings and discuss design direction with the user
   - Only when aligned, continue to parameter confirmation in step 2
   - **Fallback**: when `vision_analyze` is unavailable (missing vision provider), use the programmatic Pillow+NumPy approach in `references/programmatic-image-analysis.md` to extract color, composition, and shape data from the image file directly.
2. Before creating the task, run the interaction rules above and collect missing choices.
3. If reference images are provided as URLs, put up to 16 URLs in `image_urls`.
4. If the user attaches or references a local image file, first make sure it is available as a public URL before using this external API. The API expects public `image_urls`, not local file paths.
5. Normalize model-specific fields:
   - For `gpt-image-2-beta`: set `n=1`; omit `quality` and `resolution`.
   - For `gpt-image-2`: include `quality`; include `resolution` only when `size` is a ratio, not `auto`.
6. Create the task with `POST /v1/images/generations`.
7. Read the returned `id` as `task_id`. If no `id` exists, treat task creation as failed and report the raw non-secret error details.
8. Poll `GET /v1/tasks/{task_id}` every 3-5 seconds.
   - If polling hits a transient network/TLS timeout, do not assume the task failed. Retry the task-status query later using the same `task_id`.
   - If a 5-minute script/tool timeout kills the polling loop, make one separate lightweight status query for the same `task_id` before reporting timeout; tasks may complete after the polling process died.
9. Stop polling when:
   - `status == completed`: return every `result_data[].url`.
   - `status == failed`: report the error code/message and whether retry is reasonable.
   - 5 minutes elapsed with follow-up status still not completed: report timeout and include the `task_id` so the user can ask to continue checking.
10. In the final response, include:
   - task id
   - status
   - generated image URLs
   - selected model/options

## Python helper pattern

Use `terminal` or `execute_code` to run an actual request. Example structure:

```python
import os, time, requests

api_key = os.environ["AICODEWITH_API_KEY"]
base_url = "https://api.aicodewith.com"
headers = {"Authorization": f"Bearer {api_key}"}

body = {
    "model": "gpt-image-2",
    "prompt": prompt,
    "size": size,
    "resolution": resolution,
    "n": n,
    "quality": quality,
    "background": "auto",
}
if image_urls:
    body["image_urls"] = image_urls[:16]

resp = requests.post(
    f"{base_url}/v1/images/generations",
    headers={**headers, "Content-Type": "application/json"},
    json=body,
    timeout=60,
)
resp.raise_for_status()
task_id = resp.json()["id"]

start = time.time()
while True:
    result = requests.get(f"{base_url}/v1/tasks/{task_id}", headers=headers, timeout=60).json()
    if result.get("status") == "completed":
        urls = [item.get("url") for item in result.get("result_data", []) if item.get("url")]
        break
    if result.get("status") == "failed":
        raise RuntimeError(result.get("error"))
    if time.time() - start > 300:
        raise TimeoutError(task_id)
    time.sleep(3)
```

## Error handling

Transient network errors can happen on both create and poll requests. If task creation fails with a transport/TLS error (for example `SSLEOFError`, timeout, reset, or temporary connection failure) and no task id was returned, retry the same create request once before reporting failure. If a task id was already returned, never create a duplicate task just because polling failed; continue querying the existing task id.

Common HTTP status meanings:

- `400`: parameter error, unsupported size, prompt too long, invalid image.
- `401`: invalid API key.
- `402`: insufficient balance.
- `404`: task not found or unauthorized.
- `503`: no available channel.

Known upstream error codes:

- `content_policy_violation`: content violates policy; ask user to revise prompt.
- `invalid_parameters`: check prompt length, image size, or parameter values.
- `image_processing_error`: reference image inaccessible/unsupported/corrupt.
- `image_dimension_mismatch`: reference image dimensions mismatch request.
- `request_cancelled`: task cancelled.
- `generation_failed_no_content`: model produced no image; optimize prompt and retry.
- `service_error`, `service_unavailable`: temporary service issue; retry later.
- `generation_timeout`: generation timed out; retry later.
- `resource_exhausted`: resources busy; retry is reasonable.
- `quota_exceeded`: too frequent; wait before retrying.
- `resource_not_found`: task expired/not found.

## Pitfalls

- Never expose the raw API key.
- **Skipping reference analysis**: when the user provides a reference image and asks to create a derivative visual asset, do not jump straight to generation or parameter confirmation. Analyze the reference first (colors, composition, style), discuss findings with the user, then proceed. The user's expected workflow is: analyze → discuss → plan → confirm → generate. Skipping analysis wastes iterations and frustrates the user.
- Do not use local file paths in `image_urls`; they must be public URLs.
- Do not pass `quality` or `resolution` with `gpt-image-2-beta`.
- Do not set `n > 1` for `gpt-image-2-beta`.
- For `gpt-image-2`, only include `resolution` for ratio sizes like `1:1` or `16:9`; omit it for `auto`.
- If generating multiple images or high resolution, it can help to also mention count/resolution requirements in the prompt.
- For IP/reference-style illustration tasks, preserve concrete IP recognition anchors and reference style constraints explicitly; do not let the prompt drift into a generic product illustration. See `references/ip-style-controlled-generation.md`.
- When the user asks for an IP-only asset from a busy scene/reference image, explicitly strip every non-IP element in the prompt: no background, no ground, no game/world props, no UI, no text, no icons, and request an isolated centered transparent/alpha-ready PNG-style mascot. If the reference is only available as a local Telegram cache path and no public URL exists, use the visible image description/recognition anchors rather than passing the local path to `image_urls`. See `references/ip-only-transparent-assets.md`.
- For iterative campaign/banner variants, reuse the approved visual recipe: original IP recognition anchors + prior accepted 3D style + new business theme. Keep the style constant unless the user asks to change it; only swap the thematic objects/actions (e.g. server, recharge, analytics dashboard). See `references/session-notes-2026-06-17.md` for a concrete run pattern and transient API retry notes.
- For AI model/logo mascot banners, preserve official-logo recognition before adding cuteness. Do not substitute Codex with the OpenAI/GPT knot or a generic black terminal; use the Codex rounded-square app icon with purple-blue cloud/blob and `>_` terminal glyph. See `references/ai-model-logo-mascot-banners.md`.
- For iterative website/hero banner variants based on an approved mascot set, treat the existing IP/mascots as locked. Explicitly instruct: preserve the exact original mascot identities, silhouettes, and recognition anchors; do not redesign, replace, merge, or invent new mascots; only improve layout, background, glassmorphism UI, lighting, spacing, and composition. When the user says the design needs “more design sense/设计感”, improve hierarchy, UI framing, depth, gradients, and polish rather than swapping characters.
- For landing-page hero covers derived from busy game/fantasy reference art, first study the target template style and convert the reference into a web-usable brand hero background: reserve top navigation space, left-side low-detail text space, center/right focal mascot placement, and reduce clutter/protected-IP-like game props. See `references/landing-page-hero-banners.md`.
- For cute website/banner assets derived from a generated image, especially when the user asks to remove background/text/buttons while keeping mascot artwork, follow the cleaned-banner edit pattern in `references/token-leaderboard-banner-edits.md`.
- On Telegram and similar messaging platforms, markdown image syntax `![alt](url)` is delivered as a native image. Do not repeat the same markdown image in follow-up confirmations unless the user explicitly asks to resend/download it; otherwise it looks like multiple images were generated. For follow-ups, mention the task id/options and use a plain URL or say “上一条那张”.

## Verification

A successful run is verified only when the task status is `completed` and at least one URL exists in `result_data[].url`.
