# Using an OpenEnv environment from Claude Desktop, Cursor, or any MCP client

OpenEnv environments expose a `/mcp` endpoint that any MCP-compatible client can call directly — no Python, no training loop required. This tutorial shows how to point Claude Desktop, Cursor, and Claude.ai at a deployed Space so you can interact with an environment's tools from your favourite assistant.

```{note}
**MCP adoption in OpenEnv is still in flight.** Only `echo_env` and `finqa_env` are fully MCP-backed today. Before following this tutorial with another environment, check whether it inherits from `openenv.core.env_server.mcp_environment.MCPEnvironment`; if not, the env uses custom action types and the `/mcp` endpoint will report no tools. See [MCP Tools in OpenEnv environments](mcp-environment.md) for the full picture.
```

## Prerequisites

- An MCP-backed environment deployed to Hugging Face Spaces (e.g. `echo_env` or `finqa_env`). See the [deployment guide](../guides/deployment.md) for how to push.
- The MCP URL for your Space: `https://<owner>-<space-name>.hf.space/mcp`

  For example, if your Space is `acme/my-echo-env`, the URL is `https://acme-my-echo-env.hf.space/mcp`.

## Claude Desktop

Claude Desktop 0.10 and later supports remote MCP servers via its config file.

**1. Open the config file**

```
~/Library/Application Support/Claude/claude_desktop_config.json   # macOS
%APPDATA%\Claude\claude_desktop_config.json                        # Windows
```

**2. Add the server entry**

```json
{
  "mcpServers": {
    "openenv-echo": {
      "url": "https://acme-my-echo-env.hf.space/mcp"
    }
  }
}
```

Replace `openenv-echo` with whatever label you want to appear in Claude, and update the URL to your Space.

**3. Restart Claude Desktop**

The new server appears under the tools icon (⚙) in the chat input bar. Claude will auto-discover the available tools via `tools/list` at startup.

## Cursor

Cursor reads MCP server config from `.cursor/mcp.json` in your project directory (project-scoped) or `~/.cursor/mcp.json` (global).

**1. Create or edit `.cursor/mcp.json`**

```json
{
  "mcpServers": {
    "openenv-echo": {
      "url": "https://acme-my-echo-env.hf.space/mcp"
    }
  }
}
```

**2. Reload MCP servers**

Open the command palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) and run **MCP: Reload Servers**. The tools appear in the Cursor chat panel.

## Claude.ai (web)

Claude.ai supports MCP via custom connectors in the browser.

1. Go to **Settings → Connectors → Add custom connector**.
2. Paste your Space's MCP URL: `https://acme-my-echo-env.hf.space/mcp`
3. Save and reload the page.

The connector appears in the chat's tools sidebar and the available tools are listed automatically.

## Verifying the connection

Once connected, ask your client to list available tools — in Claude Desktop or Claude.ai you can type:

> "What tools do you have access to from openenv-echo?"

For `echo_env` you should see an `echo` tool. Call it to confirm end-to-end connectivity:

> "Use the echo tool to repeat: hello world"

Expected response: `hello world`

If the tool list is empty or the call fails, check that:

- The Space is running (not sleeping). Visit the Space URL in a browser to wake it.
- The env is MCP-backed (`echo_env` and `finqa_env` are; most others are not).
- The URL ends in `/mcp`, not just the Space root.

## How this relates to training

The `/mcp` endpoint used by external clients is the **same endpoint** that a training loop uses when you call `env.step(CallToolAction(...))` in production mode. The difference is state management:

- **External client session** — Each JSON-RPC request to `/mcp` is stateless from the client's perspective. OpenEnv tracks session IDs server-side, but there is no guaranteed episode lifecycle. This is good for interactive exploration; it is not how GRPO training works.
- **Training / orchestration** — The training harness drives the episode via the WebSocket control plane (`/ws`), calling `reset()` and `step()` explicitly. The agent's MCP tool calls go through `step()` so rewards, termination, and state are fully managed.

In short: external MCP clients let you explore and prototype with an env; training orchestrates it with precise episode control.

## Next steps

- [MCP Tools in OpenEnv environments](mcp-environment.md) — deep dive on MCP-backed environment internals
- [Deployment guide](../guides/deployment.md) — push your own environment to a Space
- [RL training with Wordle GRPO](wordle-grpo.md) — a full training loop over an MCP env
