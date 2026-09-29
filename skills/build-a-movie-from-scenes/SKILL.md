---
name: build-a-movie-from-scenes
description: "Build a multi-scene AI movie on SpicyGen: start from a library image or a character, generate scenes one after another (each continues from the last frame of the previous one), then stitch them into one video, through the SpicyGen MCP server or REST API. Use when a user asks for a longer AI video, a storyboard, a multi-scene clip, or an adult AI movie."
tags: [ai-movie, storyboard, nsfw-video-generation, adult, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicygen.ai/skills/build-a-movie-from-scenes/SKILL.md
  mcp: https://www.spicygen.ai/api/mcp
  openapi: https://www.spicygen.ai/openapi.json
  vendor: SpicyGen (Wayfinity Ltd)
  audience: adults-only
---

# Build a movie from scenes

## When to use

The user wants something longer than one clip: a sequence of scenes with the same character and setting, stitched into a single video.

## Golden path

1. `get_account` and `read_content_rules`. Plan the scenes with the user: 3 to 6 scenes of 5 to 10 seconds each, same character, same location, one action per scene. Scenes generate on SpicyMotion v3.0 Fast in HD with audio; quote the per-scene cost from `get_account.costs`.
2. First scene: `create_movie_scene` (guarded) with `source_item_id` (a library image) or `character_id`, the scene `prompt`, `seconds` and optionally a `title` and `ratio`. Omit `movie_id`; the response returns one.
3. Poll `get_job` with the returned job id until `completed`. The next scene needs the previous scene's last frame, which is extracted after completion; if `create_movie_scene` answers `previous_scene_not_ready`, wait 30 seconds and retry.
4. Next scenes: `create_movie_scene` with `movie_id` and the next prompt (no source needed). Repeat step 3.
5. `combine_movie` (guarded, no credits) with `movie_id` once every scene is completed. Present the movie URL and the editor URL (https://www.spicygen.ai/movie/<id>) where the user can reorder or re-shoot scenes in the browser.

## What to tell the user up front

- Each scene is charged when it starts; a failed scene is refunded automatically.
- Continuity comes from the last frame, so describe the same outfit and room in every prompt.

## Tools used by this skill

- `get_account`: Start here. Credits balance, age-verification state for the caller's jurisdiction (whether NSFW media is available, and the verification URL when it is not), credit costs per image, per video second and per character, rate limits, and whether the account skips approvals. Arguments: no arguments.
- `read_content_rules`: The acceptable-use rules every prompt and image is checked against (adults only, fictional characters only, no real people, no minors, no non-consent, no violence). Read this before writing prompts; a blocked prompt costs nothing and is logged; serious or repeated abuse can get the account suspended. Arguments: no arguments.
- `list_library`: The user's generated images and videos, newest first, with prompt, status, favourite flag and URLs (NSFW withheld when the age gate applies). Paginate with cursor. Arguments: `type`?: images | videos | all (Default all.); `favorites_only`?: boolean; `limit`?: integer (Default 20.); `cursor`?: string (next_cursor from a previous page.).
- `list_characters`: The user's saved characters (id, name, appearance, style, status, thumbnail). Use a character id in generate_images, generate_video and create_movie_scene for a consistent identity. Arguments: no arguments.
- `create_movie_scene` (guarded): Add a scene to a movie and start generating it on SpicyMotion v3.0 Fast (HD, audio). Omit movie_id to start a new movie; the first scene needs a source_item_id or character_id, later scenes continue from the previous scene's last frame. Costs 20 credits per second. Poll get_job with the returned job id; when it completes, call create_movie_scene again for the next scene, then combine_movie. Arguments: `movie_id`?: string (An existing movie id to append to. Omit to create a new movie.); `title`?: string (Title for a new movie.); `prompt`: string (What happens in this scene.); `seconds`?: integer (Scene length, default 5.); `source_item_id`?: string (Library image to open the movie with (first scene only).); `character_id`?: string (Character to open the movie with (first scene only).); `ratio`?: 1:1 | 16:9 | 9:16 | 4:3 | 3:4 (Aspect ratio. Default 1:1 for images, 9:16 for video.); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_job`: Status of a generation job (queued, generating, completed, failed), progress, and on completion the image or video URLs (withheld with nsfw_withheld:true when the age gate applies). Poll every 10 to 20 seconds. Arguments: `job_id`: string (A job id returned by generate_images, generate_video or create_movie_scene.).
- `combine_movie` (guarded): Stitch a movie's completed scenes into one video (no credits, but it renders on the server and is guarded so the user confirms the cut). Returns the movie URL. Arguments: `movie_id`: string; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_credit_packs`: Credit packs and prices available to this account (regional packs where they apply), with what each pack buys in images and video seconds. Arguments: no arguments.
- `get_checkout_link`: A hosted checkout URL for a credit pack. Hand the URL to the user; they pay in their browser (card, local methods or crypto depending on region). Never ask for or enter card details. Credits land in the account when the provider confirms; get_account shows the new balance. Arguments: `pack_id`: string (A pack id from get_credit_packs.).

## Setup (once per user)

1. The user signs up at https://www.spicygen.ai/signup (adults only; email confirmation is required before the first generation). Clients that support OAuth for remote MCP servers (Claude custom connectors, Smithery, the MCP Inspector) need only the URL `https://www.spicygen.ai/api/mcp`: the user signs in on SpicyGen's consent page, presses Allow, and the client receives a key. Otherwise the user creates an API key at https://www.spicygen.ai/account#api-keys. Keys look like `sg_live_…`, are shown once and can be revoked. Keep the key in an environment variable, never in chat.
2. Both surfaces expose the same tools:
   - MCP (Streamable HTTP): `https://www.spicygen.ai/api/mcp` with header `Authorization: Bearer <key>`. Claude Code: `claude mcp add --transport http spicygen https://www.spicygen.ai/api/mcp --header "Authorization: Bearer <key>"`. Cursor, Codex and any url-plus-headers client: same URL and header.
   - REST: `POST https://www.spicygen.ai/api/v1/tools/<tool_name>` with the same header and a JSON body of arguments. `GET https://www.spicygen.ai/api/v1/tools` lists tools with schemas; OpenAPI 3.1 at https://www.spicygen.ai/openapi.json.
3. Rate limit: 60 requests per minute per key. `get_account` reports the credit balance, the costs and the age-verification state.
4. Dry run first if you are building or testing an agent: a key created with the Sandbox box ticked answers every tool from fixture data, needs no credits and generates nothing. Every sandbox response carries `sandbox: true`; never present sandbox media to the user as real output.

## Rules that always apply

- Read `read_content_rules` before writing a prompt. Any depiction, implication or reference to minors in sexual situations, including "teen", school or age-play framing; age evasion is blocked. Real identifiable people: celebrities, public figures, private individuals, or uploaded photographs of real people. Deepfakes and impersonation. Non-consent, coercion, incapacitation or distress; verbal non-consent in prompts is blocked. A blocked prompt costs nothing and is logged; serious or repeated abuse can get the account suspended.
- Guarded tools (`create_character`, `generate_images`, `generate_video`, `create_movie_scene`, `combine_movie`, `delete_item`) first return `{error: "approval_required", summary, credits, approval_token}`. Show the summary and the credit cost to the user verbatim, get an explicit yes in the conversation, then call the tool again with the same arguments plus `approval_token` (15 minute expiry, bound to those exact arguments). Never approve in bulk or reuse a token for a different action. Accounts that run agents unattended can turn on skip approvals at https://www.spicygen.ai/account#api-keys; every moderation, credit and age-gate check still applies.
- Age verification: where the law requires it (United Kingdom, France, Germany, Italy, Australia, Brazil and 25 US states), NSFW media URLs are withheld (`nsfw_withheld: true`) until the user completes a one-time identity check. `get_account.age_verification` explains the state and gives the URL once it is available (after the first purchase). Explain this to the user; never try to work around it.
- Payment is a hosted checkout URL the human opens (`get_checkout_link`). Never ask for or enter card details, wallet keys or passwords.
- Everything is scoped to the account that owns the key; there is no way to read another account's data. Do not store or forward generated media outside the user's own tools.

## Costs and packs (USD, for when the user asks)

- Character: 10 credits (one identity image, reusable in every later generation).
- Image: 5 credits Standard, 10 credits HD (a character reference is always HD).
- Video, SpicyMotion v3.0: 10 credits per second Standard, 15 HD, 3 to 15 seconds.
- Video, SpicyMotion v3.0 Fast (native audio, up to 30 seconds): 15 credits per second Standard, 20 HD. Movie scenes use this model in HD.

- **100 credits**: $3.99 (EUR 3.99). About 20 standard images or 10 seconds of standard video.
- **400 credits**: $19.99 (EUR 17.99). About 80 standard images or 40 seconds of standard video.
- **2,000 credits**: $49.99 (EUR 42.99), most popular. About 400 standard images or 200 seconds of standard video.
- **10,000 credits**: $149.99 (EUR 129.99). About 2000 standard images or 1000 seconds of standard video.

Product overview for agents: https://www.spicygen.ai/llms.txt. Docs: https://www.spicygen.ai/docs/agents (every public page is also markdown: append `.md`).
