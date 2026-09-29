---
name: turn-an-image-into-a-video
description: "Animate a SpicyGen library image or a saved character into a short AI video with a motion prompt (SpicyMotion v3.0, 3 to 30 seconds, optional audio), through the SpicyGen MCP server or REST API. Use when a user asks to turn an image into a video, animate a picture, make an NSFW AI video, or generate an AI adult video clip."
tags: [image-to-video, nsfw-video-generation, video-generation, adult, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicygen.ai/skills/turn-an-image-into-a-video/SKILL.md
  mcp: https://www.spicygen.ai/api/mcp
  openapi: https://www.spicygen.ai/openapi.json
  vendor: SpicyGen (Wayfinity Ltd)
  audience: adults-only
---

# Turn an image into a video

## When to use

The user has (or wants) an image and asks for motion: animate this picture, make a clip of this character, generate a video of a scene.

## Golden path

1. `get_account`: credits, costs per second, age-verification state.
2. Pick the source: `list_library` (type images) for an existing image, or `list_characters` for a character. No image yet? Run the create-a-character-and-generate-a-set skill first, then use the resulting item id as `source_item_id`.
3. `read_content_rules`, then write a short English motion prompt: what happens, not camera jargon.
4. `generate_video` (guarded): `source_item_id` or `character_id`, `prompt`, `seconds` (default 5), `quality`, and `model` v3 (up to 15 seconds) or v3-fast (up to 30 seconds, native audio, higher per-second price). Quote the total credits, get a yes, call again with the token.
5. Poll `get_job` every 20 seconds; videos take a few minutes. Present the URL, or explain `nsfw_withheld`.
6. `favorite_item` to keep it; `delete_item` (guarded) to remove a miss.

## What to tell the user up front

- Price is per second and depends on quality and model; the approval summary states the exact total.
- The source image must be one of the user's own generations (the library) or a character; uploads of real photographs are not supported through the agent and are refused by moderation anyway.

## Tools used by this skill

- `get_account`: Start here. Credits balance, age-verification state for the caller's jurisdiction (whether NSFW media is available, and the verification URL when it is not), credit costs per image, per video second and per character, rate limits, and whether the account skips approvals. Arguments: no arguments.
- `read_content_rules`: The acceptable-use rules every prompt and image is checked against (adults only, fictional characters only, no real people, no minors, no non-consent, no violence). Read this before writing prompts; a blocked prompt costs nothing and is logged; serious or repeated abuse can get the account suspended. Arguments: no arguments.
- `list_library`: The user's generated images and videos, newest first, with prompt, status, favourite flag and URLs (NSFW withheld when the age gate applies). Paginate with cursor. Arguments: `type`?: images | videos | all (Default all.); `favorites_only`?: boolean; `limit`?: integer (Default 20.); `cursor`?: string (next_cursor from a previous page.).
- `list_characters`: The user's saved characters (id, name, appearance, style, status, thumbnail). Use a character id in generate_images, generate_video and create_movie_scene for a consistent identity. Arguments: no arguments.
- `generate_video` (guarded): Turn a library image (source_item_id) or a character (character_id) into a video with a motion prompt. Costs 10 to 20 credits per second depending on quality and model (see get_account.costs). Returns a job id; poll get_job (videos take a few minutes). Arguments: `prompt`: string (What happens in the video.); `source_item_id`?: string (A library image id (from list_library or get_job) to animate.); `character_id`?: string (A character id to animate when no source image is given.); `seconds`?: integer (Length in seconds, default 5. v3 allows up to 15, v3-fast up to 30.); `quality`?: Standard | HD (Standard or HD. HD costs more (see get_account.costs). Images with a character reference are always HD.); `ratio`?: 1:1 | 16:9 | 9:16 | 4:3 | 3:4 (Aspect ratio. Default 1:1 for images, 9:16 for video.); `model`?: v3 | v3-fast (SpicyMotion v3.0 (default) or v3.0 Fast (native audio, up to 30 seconds, higher per-second price).); `audio`?: boolean (Generate audio (v3-fast only).); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_job`: Status of a generation job (queued, generating, completed, failed), progress, and on completion the image or video URLs (withheld with nsfw_withheld:true when the age gate applies). Poll every 10 to 20 seconds. Arguments: `job_id`: string (A job id returned by generate_images, generate_video or create_movie_scene.).
- `favorite_item`: Mark or unmark a library item as a favourite. Arguments: `item_id`: string; `favorite`?: boolean (Default true; false removes the favourite.).
- `delete_item` (guarded): Delete a library image or video. Irreversible; guarded so the user confirms. Arguments: `item_id`: string; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
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
