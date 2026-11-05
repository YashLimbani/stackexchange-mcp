# Contributing

Bug reports and ideas are welcome: open an issue and include

- the client you use (VS Code, Claude Code, Antigravity CLI, ...),
- the prompt you tried and which tool was (or was not) called,
- the server log (it is written to stderr; never include your API key).

## Development

```bash
uv sync
uv run pytest -q
npx @modelcontextprotocol/inspector uv run stackexchange-mcp
```

Tests need no network: the API is mocked with real saved responses in `samples/`.

Rules of thumb: tools stay read-only, never `print()` to stdout (it carries the MCP protocol), and error messages must tell the model what to do next and never contain the API key.
