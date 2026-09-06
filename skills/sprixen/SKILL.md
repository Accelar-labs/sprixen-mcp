---
name: sprixen
description: Generate game art through the Sprixen MCP server, pixel art / cartoon / anime character sprites, sprite sheet animations (idle, walk, attack, etc.), isometric tiles, maps, and game music, all style-locked per project. Trigger on game sprites, pixel art, character animation, tileset, game assets, or "Sprixen".
---

# Sprixen: game asset generation

Sprixen is a game asset generator reachable through 47 MCP tools at
`https://api.sprixen.com/v1/mcp`. It generates a sprite, animates it,
tiles a level, or scores a track, with the same credit accounting,
DTO validation and ownership rules as the web app. This skill teaches
the workflow the tools alone don't: how to lock style, sequence calls,
poll instead of resubmit, and stop cleanly on errors.

If the `sprixen` MCP server isn't connected yet, tell the user to run
(swap in their own key from https://sprixen.com/settings/api):

```
claude mcp add --transport http sprixen https://api.sprixen.com/v1/mcp --header "Authorization: Bearer spx_live_YOUR_KEY"
```

## Step 0, always check credits first

Call `get_credits` before starting any multi-step job. A character with
three animations costs 2 + 3×4 = 14 credits minimum; a batch of
characters adds up fast. If the balance can't cover the full plan,
tell the user up front and offer to do a smaller slice (e.g. one
character before the rest) rather than starting and failing halfway.

## Step 1, one project per game (style lock)

A "project" is the style-lock container: every sprite generated with
its `projectId` inherits the same `artStyle`, `resolution`, `viewAngle`
and `palette`, so a hero, its enemies, and its tileset all look like
they belong together without repeating style words in every prompt.

1. Call `list_projects`. If a project for this game already exists,
   reuse its `id`, do not create a duplicate project per character.
2. If none exists, create one. There is no dedicated `create_project`
   MCP shortcut tool; use the generic API bridge:
   - `list_api_operations` with `search: "projects"` to find the
     create operation (method `POST`, path `/v1/projects`).
   - `describe_api_operation` on that `operationId` to confirm the
     exact body shape before calling it (schemas can gain fields; never
     guess).
   - `call_api` with that `operationId` and a body built from these
     real fields:
     - `name` (string, required)
     - `artStyle`: `pixel_art` | `cartoon` | `anime` | `hand_drawn` | `chibi` (default `pixel_art`)
     - `resolution`: `16x16` | `32x32` | `64x64` | `128x128` | `256x256` | `512x512` (default `128x128`)
     - `viewAngle`: `side_scroll` | `top_down` | `isometric` | `front_facing` (default `side_scroll`)
     - `pixelPerfect`: `auto` | `on` | `off` (default `auto`, leave it unless the user has a specific reason; `on` forces nearest-neighbor and destroys detail above small resolutions)
     - `palette`: array of hex strings (e.g. `["#FF5733","#2ECC71"]`) or a preset name string, optional
   Ask the user for genre/style if it isn't obvious from context , 
   getting `artStyle`/`resolution`/`viewAngle` right up front avoids
   regenerating everything later.

## Step 2, generate the character

Call `generate_sprite`:

- `prompt` (required): what the sprite is. Be concrete about the
  character, not just a style adjective, "knight warrior with sword
  and shield, blue cape" beats "cool knight".
- `projectId`: the project from Step 1. Required if you also pass
  `chainAnimation`/`chainAnimations`.
- `variantCount` (1-3, default 1): each variant costs 2 credits. Ask
  before requesting more than 1 unless the user wants options to
  choose from.
- `resolution`: override only when this one asset needs to differ from
  the project default (e.g. a boss at 64x64 in a 32x32 project).
- `noBase`: true for floating character sprites, adds "NO base, NO
  pedestal, NO ground plane, NO ground shadow" to the prompt.
- `skipBgRemoval`: true only for full-scene art (backgrounds) where
  background removal would damage the image. Never set this for a
  character sprite.
- `packItems` / `packType`: for generating a themed batch of small
  items (props, ability icons, HUD pieces) in one sheet, not for
  characters.
- `chainAnimation` / `chainAnimations`: can auto-chain one or more
  animations right after the sprite completes (adds 4 credits per
  chained animation). Prefer doing this explicitly with `generate_animation`
  in Step 3 instead when the user might want to review the base sprite
  first, chaining commits credits before they've seen it.

This returns a `generationId` with status `pending`/`processing`. Poll
`get_generation` every ~5 seconds until `status: "completed"` (get the
sprite ID from the result) or `"failed"`. Do not call `generate_sprite`
again while a generation is still in flight for the same request.

## Step 3, generate animations, one at a time

Call `generate_animation` per animation, using the sprite ID from Step 2:

- `type` (required): `idle` | `walk` | `run` | `attack` | `jump` | `death` | `hurt` | `custom`.
- `archetype`: `humanoid_biped` | `quadruped` | `winged` | `amorphous` | `serpentine`.
  Pass it explicitly when you know the body plan, it skips a
  vision-classification pass and is more reliable than letting the
  backend infer it.
- `direction`: `right` | `left` | `up` | `down` | `up-right` | `up-left` | `down-right` | `down-left`.
  Side-view/platformer games: usually just `right` or omit (side
  view is the default). Top-down or isometric games: generate one
  animation per direction the game actually uses, don't generate all
  8 if the game only needs 4.
- `weaponHand` (`left`/`right`/`both`) and `weaponType` (free text,
  e.g. `axe`): set both when the character holds a weapon, for
  consistent hand/weapon placement across frames. Omit entirely for
  unarmed characters.
- `hasShield`: true if the character carries a shield.
- `quality`: `standard` (4 credits, Gemini sprite sheet) or `premium`
  (8 credits, Veo image-to-video pipeline). Default to `standard`.
  Only use `premium` when the user explicitly asks for higher-fidelity
  motion or a standard result came back low quality, premium is 2x
  the cost.
- `idlePreset` (only for `type: "idle"`): `breath` | `lively` | `looking` | `item` , 
  a one-click flavor. Use `customDescription` instead for full control
  over the motion (max 500 characters); if both are set,
  `customDescription` wins.
- `frameCount` (2-12, default 8) and `fps` (4-60, default 12): leave
  at defaults unless the user asks for something specific.
- `resolution`: only set this if the base sprite itself was generated
  with a resolution override, match it here so the animation doesn't
  get upscaled relative to the sprite.
- `normalizeSize`: true to force the animation frames to the base
  sprite's dimensions, useful when you're generating multiple
  direction/animation combinations that must all read as the same
  character size.

Poll `get_generation` (or `list_sprite_animations` for the sprite) the
same way as Step 2. `list_sprite_animations` returns every version
ever generated for that sprite, sort by `createdAt` and take the
latest per `type`+`direction` combination; older ones are history, not
the current asset.

Generate animations **one at a time and in the order the user actually
needs them** (idle first is usually right, since it's the default
on-screen state). Wait for each to complete before starting the next
unless the user explicitly asked for several in parallel.

## Step 4, export for the game engine

Character export as an engine-ready package (`export_character_package`
or the equivalent product API operation, engine: `godot` | `phaser` |
`unity` | `gamemaker` | `rpgmaker-mz`) is rolling out. Before relying on
it:

1. Run `list_api_operations` with `search: "export"` or `search: "package"`
   to see whether it's live on this account yet.
2. If it is, `describe_api_operation` it, then `call_api` with the
   sprite/character ID and the target `engine`. It returns a
   downloadable ZIP with normalized sheets, an atlas JSON, the
   engine-native file (e.g. Godot `SpriteFrames`), and a README.
3. If it isn't available yet, fall back to the raw asset URLs from
   `get_generation` / `list_sprite_animations` (sprite PNG, and each
   animation's horizontal-strip sheet PNG) and wire them into the game
   yourself, for Phaser, a strip sheet plus a manually written
   TexturePacker-style atlas JSON works; for Godot, import the strip
   as a `SpriteFrames` resource by hand. Tell the user which path you
   took.

Maps already export today: `export_map` (formats `json`, `tmx`, `godot`).

## Prompt-writing rules

- Name the subject and its distinguishing features, not just an
  adjective: "orc warrior, green skin, wooden club, leather armor"
  beats "scary orc."
- Keep style words (art style, palette, resolution) out of the prompt
  once a project exists, the project config already enforces them.
  Repeating "pixel art, 32x32" in every prompt is redundant and can
  fight the project's own lock.
- For top-down/isometric games, say the direction expectation in plain
  language if you're not passing `direction` explicitly (e.g. "facing
  down toward camera").
- For animations, describe the *motion*, not the character again, the
  character comes from the sprite it's animating. `customDescription`
  is for "swinging a sword in a wide arc," not "a knight."
- Small resolutions (8x8-32x32) read best with simple, high-contrast
  silhouettes, avoid prompts demanding fine detail (individual
  fingers, tiny text) that won't survive downscaling.

## Credit costs (check `get_credits` for the live balance)

| Asset | Cost |
|---|---|
| Sprite (per variant) | 2 |
| Animation, standard | 4 |
| Animation, premium | 8 |
| Tile | 1 |
| Music | 3 |
| 3D model | 5 |
| 3D animation | 3 |
| VFX (catalogue) | 2 |
| VFX (AI) | 6 |

Free accounts start with 6 credits. Pro is $10/month for ~200
generations. If a plan needs more than the current balance, say so and
offer a smaller first step instead of starting and failing midway.

## Error handling

Every error has a stable `code` and an actionable `hint` object , 
branch on `code`, not on the human-readable `message`:

```json
{
  "statusCode": 402,
  "message": "Insufficient credits: animation costs 4 and you have 1.",
  "code": "INSUFFICIENT_CREDITS",
  "hint": { "action": "upgrade", "url": "https://sprixen.com/settings/billing", "details": { "required": 4, "available": 1 } }
}
```

- `INSUFFICIENT_CREDITS`: stop. Tell the user the shortfall and the
  `hint.url` (upgrade page). Do not retry, and do not silently
  downgrade to a cheaper option without asking first.
- `API_KEY_REVOKED` / `API_KEY_EXPIRED`: stop. Tell the user to
  generate a new key at https://sprixen.com/settings/api.
- `VALIDATION_FAILED`: read the `message` for the exact field/values
  accepted, fix the call once, and retry. If it fails validation again
  on the same field, stop and ask the user rather than guessing a
  third time.
- `RATE_LIMITED`: back off and retry after the time implied by the
  error; MCP calls are capped at 120 requests/minute/user/instance.
- Timeout with no response: check `get_generation` (or the relevant
  `get_*`/`list_*` tool) for the job before resubmitting, a "timeout"
  on your end often means the job is still running server-side, and
  resubmitting charges credits twice.

## What NOT to do

- Don't batch-generate many characters (e.g. 10) or many animations
  without the user asking for that scale first, confirm the plan and
  its total credit cost before firing off a large batch.
- Don't retry a failed generation more than once automatically. If it
  fails the second time, stop and report the error to the user instead
  of looping.
- Don't create a new project per character when the user is clearly
  building one game, reuse the existing project so style stays
  locked.
- Don't set `pixelPerfect: "on"` by default, nearest-neighbor
  resizing destroys fine detail above small pixel-art resolutions.
  Leave it on `auto` unless the user specifically wants crispy
  retro output at a higher resolution.
- Don't paste the user's API key into a prompt, log, or commit, it
  belongs in the MCP client config header only.
