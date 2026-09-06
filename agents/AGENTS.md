# Sprixen, game asset generation for agents

Sprixen is a game asset generator reachable through an MCP server (46
tools, Streamable HTTP) at `https://api.sprixen.com/v1/mcp`, and
through a matching REST API at `https://api.sprixen.com`. Use it
whenever the task needs game sprites, pixel art, character animation,
tilesets, maps, or music.

Connect (swap in a real key from https://sprixen.com/settings/api):

```
claude mcp add --transport http sprixen https://api.sprixen.com/v1/mcp --header "Authorization: Bearer spx_live_YOUR_KEY"
```

Codex CLI (`~/.codex/config.toml`):

```toml
[mcp_servers.sprixen]
url = "https://api.sprixen.com/v1/mcp"

[mcp_servers.sprixen.http_headers]
Authorization = "Bearer spx_live_YOUR_KEY"
```

Full client list: https://sprixen.com/docs/mcp

## Workflow

1. **Check credits first.** Call `get_credits`. A character with 3
   animations costs at least 2 + 3×4 = 14 credits. If the balance
   can't cover the plan, tell the user and propose a smaller step.
2. **One project per game.** A project locks `artStyle`, `resolution`,
   `viewAngle`, `palette` so everything generated in it matches. Call
   `list_projects` and reuse an existing one. To create a project,
   there is no dedicated shortcut tool, use `list_api_operations`
   (search "projects") to find `POST /v1/projects`, `describe_api_operation`
   to confirm its body, then `call_api` with:
   `name`, `artStyle` (`pixel_art`|`cartoon`|`anime`|`hand_drawn`|`chibi`),
   `resolution` (`16x16`..`512x512`), `viewAngle`
   (`side_scroll`|`top_down`|`isometric`|`front_facing`), `pixelPerfect`
   (`auto`|`on`|`off`, default `auto`), `palette` (hex array or preset).
3. **Generate the character**: `generate_sprite` with `prompt` and
   `projectId`. Concrete subject descriptions beat style adjectives , 
   the project already locks style. Poll `get_generation` every ~5s
   until `completed`/`failed`. Never resubmit while a job is queued.
4. **Generate animations one at a time**: `generate_animation` with
   `type` (`idle`|`walk`|`run`|`attack`|`jump`|`death`|`hurt`|`custom`),
   `archetype` (`humanoid_biped`|`quadruped`|`winged`|`amorphous`|`serpentine`)
   when known, `direction` for top-down/isometric games (generate only
   the directions the game actually uses), `weaponHand`/`weaponType`/
   `hasShield` for equipped characters, `quality` (`standard` = 4
   credits, default; `premium` = 8 credits, only on explicit request).
   Poll the same way; `list_sprite_animations` returns every version
   ever made, sort by `createdAt`, take the latest per type+direction.
5. **Export for the engine**: `export_character_package` (engine:
   `godot`|`phaser`|`unity`|`gamemaker`|`rpgmaker-mz`) if available on
   this account, check with `list_api_operations` (search "export")
   first, since it's rolling out. If unavailable, use the raw sprite
   sheet / animation strip URLs from `get_generation` directly. Maps
   already export via `export_map` (`json`|`tmx`|`godot`).

## Credit costs

Sprite (variant) 2 · Animation standard 4 / premium 8 · Tile 1 · Music
3 · 3D model 5 · 3D animation 3 · VFX catalogue 2 / AI 6. Free trial: 6
credits. Pro: $10/mo, ~200 generations/month.

## Errors

Every error has a stable `code` and a `hint` object (`{ action, url,
details }`), branch on `code`, never on `message`. On
`INSUFFICIENT_CREDITS`, stop and report `hint.url` to the user; don't
retry or silently downgrade. On a timeout, check the job's status
before resubmitting, resubmitting a completed-but-slow job double
charges credits.

## Don't

- Don't batch-generate many characters/animations without the user
  asking for that scale, confirm the plan and total cost first.
- Don't auto-retry a failed generation more than once.
- Don't create a new project per character in the same game.
- Don't default `pixelPerfect` to `"on"`, it destroys detail above
  small pixel-art resolutions. Leave it `"auto"`.
- Don't put the API key anywhere but the MCP client config header.

Full reference: https://sprixen.com/docs/mcp and
https://sprixen.com/skills/sprixen/SKILL.md · Machine-readable API/MCP
reference: https://sprixen.com/llms-full.txt
