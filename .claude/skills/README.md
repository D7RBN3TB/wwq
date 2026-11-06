# Montage skills

Project-level Claude Code skills and agent for [Montage](https://montage.app),
adapted from the MIT-licensed montage plugin
(https://github.com/ukumi-ai/montage-plugin, v0.2.1; see `MONTAGE-LICENSE`).

| Path | What it does |
| --- | --- |
| `skills/ingest-video/` | Ingest a video from a Google Drive / public URL and poll it through the pipeline |
| `skills/create-moment/` | Turn an ingested video into an editable moment (clip) and return the editor link |
| `../agents/clip-scout.md` | Read-only agent that proposes clip candidates with timestamps |

The skills call tools on the Montage MCP server, configured in the repo-root
`.mcp.json` (`https://mcp.montage.app/mcp`). You need to authorize that server
(OAuth) the first time you use it.
