# Sprixen MCP server

Sprixen is an AI game asset generator with a native [Model Context Protocol](https://modelcontextprotocol.io) server. Agents in Claude Code, Cursor, Codex, Windsurf, VS Code, Claude Desktop and Gemini CLI can create a style-locked project, generate character sprites, animate them, generate tiles, maps, music and VTT effects, and export an engine-ready package for Godot, Phaser, Unity, GameMaker or RPG Maker MZ.

- Endpoint: `https://api.sprixen.com/v1/mcp` (Streamable HTTP)
- Auth: `Authorization: Bearer spx_live_...` (create a key at https://sprixen.com/settings/api, 6 free credits at signup)
- Setup for every client: https://sprixen.com/docs/mcp
- Full API reference: https://sprixen.com/docs and https://sprixen.com/llms-full.txt
- Registry entry: `com.sprixen/sprixen` in the official MCP Registry (`server.json` in this repo)

## Quick start

Claude Code:

```bash
claude mcp add --transport http sprixen https://api.sprixen.com/v1/mcp \
  --header "Authorization: Bearer spx_live_YOUR_KEY"
```

Cursor (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "sprixen": {
      "url": "https://api.sprixen.com/v1/mcp",
      "headers": { "Authorization": "Bearer spx_live_YOUR_KEY" }
    }
  }
}
```

Codex CLI (`~/.codex/config.toml`):

```toml
[mcp_servers.sprixen]
url = "https://api.sprixen.com/v1/mcp"

[mcp_servers.sprixen.http_headers]
Authorization = "Bearer spx_live_YOUR_KEY"
```

## Teach your agent

The `skills/sprixen/SKILL.md` file is a Claude Code skill with the full workflow (check credits, one project per game for style lock, generate a character, generate animations one at a time and poll, export with `export_character_package`). `agents/AGENTS.md` is the same guidance for Codex and generic agents; `agents/sprixen.mdc` is a Cursor rule.

```bash
mkdir -p ~/.claude/skills/sprixen && curl -fsSL https://sprixen.com/skills/sprixen/SKILL.md -o ~/.claude/skills/sprixen/SKILL.md
```

## Tools

47 tools: `get_credits`, `list_projects`, `generate_sprite`, `get_generation`, `list_sprites`, `generate_animation`, `list_sprite_animations`, `export_character_package`, `generate_tile`, `list_tile_packs`, `create_map` / `list_maps` / `export_map`, `generate_music`, 3D model and animation tools, VFX tools, AI Game Builder and Mod Builder tools, plus `list_api_operations` / `describe_api_operation` / `call_api` to reach every product endpoint (167 operations) with the same validation, ownership and credit rules as the app.

Credits: sprite 2, animation 4 (premium 8), tile 1, music 3, 3D model 5, character package export free. Pro is $10/month for 200 generations.

## Errors are actionable

Every error body carries a stable `code` and a `hint` with the next step, for example:

```json
{
  "statusCode": 402,
  "code": "INSUFFICIENT_CREDITS",
  "message": "Insufficient credits: this generation costs 2 and you have 0.",
  "hint": { "action": "upgrade", "url": "https://sprixen.com/settings/billing" }
}
```

## License

The files in this repository are MIT licensed. The Sprixen service itself is governed by https://sprixen.com/terms.
