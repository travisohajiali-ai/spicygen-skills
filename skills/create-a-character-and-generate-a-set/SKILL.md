---
name: create-a-character-and-generate-a-set
description: "Create a reusable AI character from a text description on SpicyGen (an AI adult image and video generator, 18+), then generate a set of images of that character in different scenes, through the SpicyGen MCP server or REST API. Use when a user asks for an AI character, a consistent AI model or persona, AI adult images of the same person, or an NSFW AI image generator."
tags: [ai-characters, nsfw-image-generation, image-generation, adult, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicygen.ai/skills/create-a-character-and-generate-a-set/SKILL.md
  mcp: https://www.spicygen.ai/api/mcp
  openapi: https://www.spicygen.ai/openapi.json
  vendor: SpicyGen (Wayfinity Ltd)
  audience: adults-only
---

# Create a character and generate a set of images

## When to use

The user wants images of one consistent fictional adult: an original character, a persona for a series, or a set of images in different settings. Characters keep the identity across every later image, video and movie scene.

## Golden path

1. `get_account`: confirm credits and the age-verification state. If credits are short, `get_credit_packs` then `get_checkout_link` and hand the user the URL.
2. `read_content_rules`: adults only, fictional only, no real people. Turn the user's description into an English appearance description that names an adult age ("a woman in her thirties").
3. `list_characters`: reuse an existing character when one matches.
4. `create_character` (guarded, 10 credits): name, appearance, style. Show the approval summary, get a yes, call again with the token. Then poll `get_character` every 15 seconds until `status` is `ready` (about a minute). If it fails, the credits are refunded automatically.
5. `generate_images` (guarded) with `character_id`, a scene prompt, `count` up to 10 and a ratio. Images with a character reference are always HD. Quote the total before asking for approval.
6. Poll `get_job` for each job id every 10 to 20 seconds until `completed`; present the URLs (or explain `nsfw_withheld` and the verification URL from `get_account`).
7. Variations: call `generate_images` again with the same `character_id` and a changed prompt. `favorite_item` for the ones the user likes; `list_library` to review everything.

## What to tell the user up front

- Every image is moderated; a refused prompt costs nothing.
- Costs are per image and are quoted before anything is charged.

## Tools used by this skill

- `get_account`: Start here. Credits balance, age-verification state for the caller's jurisdiction (whether NSFW media is available, and the verification URL when it is not), credit costs per image, per video second and per character, rate limits, and whether the account skips approvals. Arguments: no arguments.
- `read_content_rules`: The acceptable-use rules every prompt and image is checked against (adults only, fictional characters only, no real people, no minors, no non-consent, no violence). Read this before writing prompts; a blocked prompt costs nothing and is logged; serious or repeated abuse can get the account suspended. Arguments: no arguments.
- `list_characters`: The user's saved characters (id, name, appearance, style, status, thumbnail). Use a character id in generate_images, generate_video and create_movie_scene for a consistent identity. Arguments: no arguments.
- `create_character` (guarded): Create a reusable character from a text description (10 credits). Generates the identity image; poll get_character until status is ready (about a minute). The description goes through the same prompt moderation as the website. Arguments: `name`: string (Display name, up to 40 characters.); `appearance`: string (Appearance: body, face, hair, age (adults only), clothing, setting. 10 to 2000 characters.); `style`?: Photorealistic | Studio | Hentai (Anime) | 3D | Cartoon (Art style. Default Photorealistic.); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_character`: One character with its recent generations (images and videos). NSFW media URLs are withheld when the account's jurisdiction requires age verification the user has not completed. Arguments: `character_id`: string (An id from list_characters.).
- `generate_images` (guarded): Generate 1 to 10 images from a prompt (5 credits each Standard, 10 HD; a character reference forces HD). Returns job ids; poll get_job. Prompts are moderated exactly as on the website; call read_content_rules first. Arguments: `prompt`: string (The scene, up to 2000 characters.); `character_id`?: string (Generate this saved character (from list_characters). Omit for a new subject.); `count`?: integer (Number of images, default 1.); `quality`?: Standard | HD (Standard or HD. HD costs more (see get_account.costs). Images with a character reference are always HD.); `ratio`?: 1:1 | 16:9 | 9:16 | 4:3 | 3:4 (Aspect ratio. Default 1:1 for images, 9:16 for video.); `style`?: Photorealistic | Studio | Hentai (Anime) | 3D | Cartoon (Art style. Default Photorealistic.); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_job`: Status of a generation job (queued, generating, completed, failed), progress, and on completion the image or video URLs (withheld with nsfw_withheld:true when the age gate applies). Poll every 10 to 20 seconds. Arguments: `job_id`: string (A job id returned by generate_images, generate_video or create_movie_scene.).
- `list_library`: The user's generated images and videos, newest first, with prompt, status, favourite flag and URLs (NSFW withheld when the age gate applies). Paginate with cursor. Arguments: `type`?: images | videos | all (Default all.); `favorites_only`?: boolean; `limit`?: integer (Default 20.); `cursor`?: string (next_cursor from a previous page.).
- `favorite_item`: Mark or unmark a library item as a favourite. Arguments: `item_id`: string; `favorite`?: boolean (Default true; false removes the favourite.).
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
