# sbx calls mcp server

```bash
git clone https://codeberg.org/ai-apocalypse-survival-kit/mcp-scripts.git
```


## Star the MCP server

```bash
cd mcp-scripts
docker compose -f compose.dev.yml up --build -d
```

### Testing the MCP server


Type the following command to see the logs of the MCP server:
```bash
docker compose -f compose.dev.yml logs
```

Then you can retrieve the URL of the UI of MCP Inspector, something like this:
```
http://0.0.0.0:6274/?MCP_PROXY_AUTH_TOKEN=d9390f5751ef6a8152667c89a1d471d4abefca30bd34b6886feef0319a0f5cac
```
