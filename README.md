# stackexchange-mcp

An [MCP](https://modelcontextprotocol.io) server that lets your AI assistant search **Stack Overflow** and other **Stack Exchange** sites, read questions with their best answers, and ground its debugging help in real, voted answers instead of guessing.

- **Read-only:** it never posts, votes or edits anything.
- **Model-friendly output:** short summaries, Markdown instead of HTML, code blocks preserved, long posts truncated with a link.
- **Respects the API:** optional API key, backoff handling, request spacing, retries, caching and quota tracking.
- **Clear errors:** every failure tells the model what happened and what to try next.

> Not affiliated with Stack Exchange, Inc. Content returned by the API is licensed CC BY-SA (3.0 or 4.0 depending on the post). The server returns the author, link and license so assistants can attribute it.

## What it provides

[project]
name = "mcp-server"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
"httpx>=0.28.1",
"mcp[cli]>=1.0,<2",
"pydantic>=2.14.2",
]

[dependency-groups]
dev = [
"pytest>=9.2.0",
]

### Tools

| Tool                      | Use it to                                                                                                             |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `search_questions`        | Find discussions about a topic. Filter by tags, site, sort order, accepted answers only.                              |
| `get_question`            | Read one question with its best answers (accepted answer first).                                                      |
| `search_by_error_message` | Find threads for a pasted error or stack trace. File paths, line numbers and addresses are stripped before searching. |
| `search_tags`             | Find exact tag names (so the assistant does not guess tags).                                                          |
| `get_tag_overview`        | Get a technology's summary, question count and top-voted questions.                                                   |
| `get_quota_status`        | See the remaining API quota and any active rate-limit wait.                                                           |

### Resources

| URI                                | Contents                                                 |
| ---------------------------------- | -------------------------------------------------------- |
| `stackexchange://sites`            | Site slugs you can pass as `site` (cached for 24 hours). |
| `stackexchange://help/search-tips` | Guidance on effective searches.                          |
| `stackexchange://quota`            | Live quota snapshot (JSON).                              |

### Prompts

| Prompt               | What it does                                                                             |
| -------------------- | ---------------------------------------------------------------------------------------- |
| `debug_error`        | Searches for your error, reads the best two threads, explains the likely fix with links. |
| `compare_approaches` | Compares two approaches using highly voted discussions.                                  |
| `tag_primer`         | Builds a beginner overview of a technology from its tag.                                 |

## Quick start

### 1. Get a free API key (recommended)

The server works without a key, but the anonymous quota is small: the API reported **300 requests per day without a key and 10,000 with one**. Register an app on [Stack Apps](https://stackapps.com/apps/oauth/register) to get a key; you do not need OAuth for read-only use.

### 2. Install

You need [uv](https://docs.astral.sh/uv/). The server runs straight from PyPI:

```bash
uvx stackexchange-mcp
```

(That starts a stdio server and waits; you normally let your AI app launch it using one of the configs below.)

### 3. Connect your AI app

Use the full path to `uvx` if your app cannot find it (`which uvx`).

**VS Code (GitHub Copilot, Agent mode)** - `.vscode/mcp.json`:

```json
{
  "servers": {
    "stackexchange": {
      "type": "stdio",
      "command": "uvx",
      "args": ["stackexchange-mcp"],
      "env": { "STACKEXCHANGE_KEY": "${input:se_key}" }
    }
  },
  "inputs": [
    {
      "type": "promptString",
      "id": "se_key",
      "description": "Stack Exchange API key",
      "password": true
    }
  ]
}
```

**Antigravity CLI** - `.agents/mcp_config.json` in your project (or `~/.gemini/config/mcp_config.json` globally):

```json
{
  "mcpServers": {
    "stackexchange": {
      "command": "uvx",
      "args": ["stackexchange-mcp"],
      "env": { "STACKEXCHANGE_KEY": "your-key" }
    }
  }
}
```

Start `agy` and run `/mcp` to confirm it is connected.

**Claude Code:**

```bash
claude mcp add --env STACKEXCHANGE_KEY=your-key stackexchange -- uvx stackexchange-mcp
```

**Claude Desktop (macOS/Windows) and Cursor** - same JSON shape as Antigravity (`mcpServers`), in `claude_desktop_config.json` or `~/.cursor/mcp.json`.

> Keep your key out of files you commit.

## Configuration

| Variable               | Default         | Purpose                                                        |
| ---------------------- | --------------- | -------------------------------------------------------------- |
| `STACKEXCHANGE_KEY`    | none            | API key. Raises the daily quota.                               |
| `SE_DEFAULT_SITE`      | `stackoverflow` | Site used when a tool call does not specify one.               |
| `SE_CACHE_TTL_SECONDS` | `300`           | How long identical requests are cached (`0` disables caching). |
| `SE_MAX_BODY_CHARS`    | `2000`          | Maximum characters of any question or answer body returned.    |

## Rate limits and errors

- Identical requests within the cache window cost no quota.
- If the API sends a `backoff` value, the server waits it out (up to 10 seconds) before the next call; longer waits are reported instead of blocking.
- Requests are spaced at least 0.2 seconds apart.
- Timeouts, network errors and 5xx responses are retried twice with exponential delay (0.5 s, then 1 s).
- When the daily quota is exhausted the server stops calling the API until the next UTC day.

| Situation                         | What the assistant is told                               |
| --------------------------------- | -------------------------------------------------------- |
| Invalid input (bad site, bad tag) | Which parameter is wrong and how to fix it.              |
| Question or tag not found         | It may be deleted or on another site; use `search_tags`. |
| No search results                 | Broaden the query, remove tags, or use `search_tags`.    |
| Rate limited                      | How long to wait.                                        |
| Quota exhausted                   | Set `STACKEXCHANGE_KEY` or wait for the daily reset.     |
| Invalid API key                   | Check or unset `STACKEXCHANGE_KEY`.                      |
| Stack Exchange unavailable        | Usually temporary; try again shortly.                    |

The API key is never written to logs or error messages.

## Example prompts

You do not need to name a tool. Things that should make the assistant use this server:

- "What's the difference between `asyncio.sleep` and `time.sleep`? Check Stack Overflow."
- "I get `AttributeError: 'NoneType' object has no attribute 'foo'` - find help on Stack Overflow." (paste the traceback)
- "Is there a Stack Overflow tag for testing async code with pytest?"
- "I'm new to python-asyncio. Give me an overview from Stack Overflow."
- "Find recent discussions about Docker networking on Server Fault."

## Tested with

Results for each prompt are in [TESTING.md](TESTING.md).

- **Antigravity CLI:** connects and lists all 6 tools. In a 10-prompt run, the model picked the expected tool without being told its name in 9 of 10 prompts (one prompt still needs a re-run) and chose sensible arguments such as `site`, `sort` and `accepted_only`. It correctly used no tool for an unrelated question.
- **VS Code (Copilot Agent mode):** the server starts and 6 tools are discovered; tool-call results are not recorded yet.
- **MCP Inspector:** connects and lists the server's tools (re-check all resources and prompts before release).

## Troubleshooting

- **Server "anonymous (low quota)" in the log:** the key did not reach the server. Put `STACKEXCHANGE_KEY` in the `env` block of your client's config.
- **Command not found:** use the absolute path of `uvx` in the config.
- **VS Code shows no tools:** use a normal Copilot **Agent mode** chat session in a workspace that contains `.vscode/mcp.json`; some session types keep their own separate server list.
- **The assistant never calls a tool:** ask it explicitly once ("search Stack Overflow for ...") and check the tool list in your client.
- **Logs:** the server writes logs to stderr (never stdout, which carries the protocol). In VS Code, open the Output panel and choose the server.

## Development

```bash
git clone https://github.com/YashLimbani/stackexchange-mcp
cd stackexchange-mcp
uv sync
uv run pytest -q
npx @modelcontextprotocol/inspector uv run stackexchange-mcp
```

Tests need no network: the API is mocked using real saved responses in `samples/`. See [DESIGN.md](DESIGN.md) for the design and [CONTRIBUTING.md](CONTRIBUTING.md) for how to help.

## License

MIT for the code (see [LICENSE](LICENSE)). Stack Exchange content is CC BY-SA; attribute the author and link when you reuse it.
