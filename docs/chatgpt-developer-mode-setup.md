# ChatGPT Developer Mode Integration Guide

This guide walks you through hosting the MCP Picnic server so that ChatGPT (Developer Mode) can reach it securely over HTTPS.

## 1. Build the server

```bash
npm install
npm run build
```

## 2. Configure credentials

Create a `.env` file (or export environment variables before starting the server):

```bash
cp .env.example .env
# edit .env with your Picnic login details
```

| Variable | Required | Purpose |
| --- | --- | --- |
| `PICNIC_USERNAME` | ✅ | Picnic account email |
| `PICNIC_PASSWORD` | ✅ | Picnic password |
| `PICNIC_COUNTRY_CODE` | ➖ | `NL` (default) or `DE` |
| `ENABLE_HTTP_SERVER` | ✅ | Must be `true` for ChatGPT access |
| `HTTP_HOST` | ➖ | Usually `0.0.0.0` when tunnelling |
| `HTTP_PORT` | ➖ | External port, `3000` by default |

> Never commit your populated `.env` file. Treat Picnic credentials as secrets.

## 3. Start the HTTP transport

```bash
node ./bin/mcp-server.js \
  --enable-http \
  --http-host 0.0.0.0 \
  --http-port 3000
```

On success you should see `MCP StreamableHTTP server running on http://0.0.0.0:3000/mcp`.

## 4. Provide an HTTPS endpoint

ChatGPT only connects to HTTPS URLs. Pick one of the following options:

1. **Cloudflare Tunnel** (fastest) – `cloudflared tunnel --url http://localhost:3000`
2. **ngrok / LocalTunnel** – exposes a temporary HTTPS URL
3. **Reverse proxy** (Nginx/Caddy/Traefik) – terminate TLS and forward to `localhost:3000`
4. **Cloud host** – deploy the server behind HTTPS (Fly.io, Render, Vercel Edge, etc.)

Keep `/mcp` open for POST / GET / DELETE and leave `/health` for monitoring. Do not expose this server without authentication and rate limiting.

## 5. Test connectivity

```bash
curl -i https://YOUR-DOMAIN.example/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"ping","method":"initialize","params":{"protocolVersion":"2025-03-26"}}'
```

A valid response includes status `200`, an `mcp-session-id` header, and a JSON body.

## 6. Enable ChatGPT Developer Mode

1. Log in to [chatgpt.com](https://chatgpt.com/)
2. Profile icon → **Settings** → **Connectors** → **Advanced**
3. Toggle **Developer mode** (beta, Plus/Pro/Business/Education accounts only)
4. In **Connectors**, click **Add Server** → **Remote MCP server**
5. Fill in:
   - **Label**: `Picnic`
   - **Server URL**: `https://YOUR-DOMAIN.example/mcp`
   - **Description**: optional
   - **Approval policy**: keep on **Always** until you trust the flow
6. Save – the connector now appears under *Developer Mode* inside the chat composer.

## 7. Use the connector in ChatGPT

- Start a new conversation, open the `+` menu, choose **Developer Mode**, then select `Picnic`
- The first call (`mcp_list_tools`) imports tool definitions; approve it
- Review every subsequent approval carefully, especially for write operations (`picnic_add_to_cart`, etc.)
- Once comfortable, you can disable approvals per tool from **Settings → Connectors → Picnic → Require approval**

## 8. Manage secrets and security

- Rotate Picnic credentials periodically and revoke tunnels when not in use
- Monitor server logs for suspicious tool usage or prompt-injection attempts
- Rate-limit your public endpoint and keep the host patched
- Never grant persistent approvals unless you fully trust the environment

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Missing mcp-session-id header` | Ensure ChatGPT request includes the header; the server handles this automatically in normal flow |
| `HTTPS required` | Use a tunnel or proxy that provides TLS termination |
| `401 Picnic authentication failure` | Re-check credentials in `.env` |
| Long-running requests dropped | Increase proxy timeout; SSE connections must stay open |

Once the server is reachable via HTTPS, ChatGPT Developer Mode can use it just like Claude, Continue, or any other MCP client.
