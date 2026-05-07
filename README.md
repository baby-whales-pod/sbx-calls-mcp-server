# sbx calls mcp server

## We need a MCP server with tools

Clone this project:
```bash
git clone https://codeberg.org/ai-apocalypse-survival-kit/mcp-scripts.git
```
> It's a side project I did to test some coding agents.

### Star the MCP server

```bash
cd mcp-scripts
docker compose -f compose.dev.yml up --build -d
# or if already build
docker compose -f compose.dev.yml up -d
```

### You can test the MCP server with MCP Inspector tool (optional)


Type the following command to see the logs of the MCP server:
```bash
docker compose -f compose.dev.yml logs
```

Then you can retrieve the URL of the UI of MCP Inspector, something like this:
```
http://0.0.0.0:6274/?MCP_PROXY_AUTH_TOKEN=d9390f5751ef6a8152667c89a1d471d4abefca30bd34b6886feef0319a0f5cac
```

## Create a sbx sandboxes with an agent that can use the MCP tools

At the root of the directory, run the following command to create a sbx sandbox with the kit configuration:

```bash
sbx create claude --kit ./kit .  
sbx run claude-sbx-calls-mcp-server
```

Type `/mcp` to see the list of available MCP tools.

You can also ask:
```
What MCP tools can you use?
```

Then
```
Ok, get system information
```