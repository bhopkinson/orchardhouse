# mcpproxy On The Edge Tunnel

mcpproxy is a Portainer stack in `portainer/stacks/mcp-proxy`. Traefik publishes one hostname. Configured MCP servers are not on that hostname.

```text
mcp.orchardhouse.cc
```

The web UI is `https://mcp.orchardhouse.cc/ui/`. The MCP endpoint is `https://mcp.orchardhouse.cc/mcp`.

## How Traffic Flows

Browsers:

```text
Browser
Cloudflare Access Microsoft login
Cloudflare Tunnel
cloudflared
Traefik
mcpproxy /ui and /api
```

MCP clients:

```text
MCP client
Cloudflare Access service token
Cloudflare Tunnel
cloudflared
Traefik
mcpproxy /mcp
```

The container publishes no host ports. The only published route is the Traefik host rule for `mcp.${DOMAIN}`.

## Cloudflare Access For The Web UI

The web UI and REST API are covered by the existing wildcard Access application:

```text
Hostname: *.orchardhouse.cc
Policy: Allow your personal Microsoft account
Identity provider: Microsoft Personal OIDC
```

You do not need a separate Access application for `mcp.orchardhouse.cc` while that wildcard application is in place.

Do not add a Bypass policy for this hostname. A bypass would publish the configured MCP tools on the internet.

## Cloudflare Access For MCP Clients

MCP clients cannot complete the Microsoft browser login. They use a service token, and only on the MCP paths.

Create a Cloudflare Access service token:

1. Go to `Access` -> `Service Auth` -> `Service Tokens`.
2. Create a token named `mcp-proxy-clients`.
3. Copy the `Client ID`.
4. Copy the `Client Secret`.

This is a different token from `github-actions-portainer-webhook`. The GitHub Actions token can redeploy stacks. It must not be able to call MCP tools.

Create a more specific self-hosted Access application:

```text
Hostname: mcp.orchardhouse.cc
Path: /mcp*
Policy action: Service Auth
Include: mcp-proxy-clients
```

Do not create a normal Allow policy on this application.

`/mcp*` has to cover `/mcp`, `/mcp/all`, `/mcp/call`, and `/mcp/code`. Those paths serve the configured tools. If they stay on the wildcard application only, a service token is not accepted and a browser login is required instead.

`/ui` and `/api` stay on the wildcard Microsoft login. The service-token application must not include those paths.

## mcpproxy Credentials

Cloudflare Access admits the request. mcpproxy then requires its own credential. The seed config sets `require_mcp_auth` to `true` before the first boot. Leave that on. With it off, anyone who reaches the origin can call the configured tools.

The admin API key is for the web UI. Create MCP client credentials in the mcpproxy Clients hub. Those `mcp_cli_` credentials are valid on MCP endpoints only. Do not put the admin API key in Cursor or any other MCP client.

A client calls:

```text
https://mcp.orchardhouse.cc/mcp
```

with these headers:

```text
CF-Access-Client-Id: <mcp-proxy-clients client ID>
CF-Access-Client-Secret: <mcp-proxy-clients client secret>
Authorization: Bearer <mcp_cli credential>
```

## Configured MCP Servers Stay Private

Traefik on the edge stack only routes containers on `orchard_proxy`. mcpproxy is on that network so Traefik can reach it. It is also on `mcp_internal`.

Put every configured MCP server on `mcp_internal` only.

- Do not attach those containers to `orchard_proxy`.
- Do not add Traefik labels to them.
- Do not publish their ports on the host.

mcpproxy is the only thing that should call them. The server image does not mount the Docker socket, and it does not include Node or Python, so it cannot spawn local `npx` or `docker` MCP servers itself.

New servers stay quarantined until they are approved in the web UI. That is the default, and the seed config keeps `quarantine_enabled` set.

## Portainer Stack Environment

Set these on the Portainer stack. Do not commit real values.

```text
DOMAIN=orchardhouse.cc
MCPPROXY_API_KEY=<openssl rand -hex 32>
MCPPROXY_TRUSTED_PROXIES=<orchard_proxy subnet>
```

`MCPPROXY_API_KEY` is the web UI admin key. Keep a copy. The image is distroless, so an auto-generated key is awkward to read back out of the container.

`MCPPROXY_TRUSTED_PROXIES` must be the `orchard_proxy` subnet and nothing wider. On the Docker host:

```bash
docker network inspect orchard_proxy --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

Trusting a broader range would let a container on `mcp_internal` spoof forwarded headers.

The state volume is mounted at `/root/.mcpproxy`. Do not move it. A different data directory is ignored and the process replaces the config, including the API key, on every boot.

The seed file is copied only when that volume has no `mcp_config.json` yet. Later edits in the web UI persist on the volume and are not overwritten by a redeploy.

## GitHub Actions

After the stack exists in Portainer, enable its stack webhook. Portainer may show a local URL. Keep the path and token, and use the webhook hostname:

```text
https://portainer-webhook.orchardhouse.cc/api/stacks/webhooks/<webhook-token>
```

Add this GitHub secret:

```text
PORTAINER_STACK_MCP_PROXY_WEBHOOK=https://portainer-webhook.orchardhouse.cc/api/stacks/webhooks/<webhook-token>
```

The workflow uses the existing `CF_ACCESS_CLIENT_ID` and `CF_ACCESS_CLIENT_SECRET` secrets, which belong to the Portainer webhook service token. Pushes that change `portainer/stacks/mcp-proxy/**` validate the Compose file and then call that webhook.

## Check After Deploy

- `https://mcp.orchardhouse.cc/ui/` prompts for Microsoft login and does not load anonymously.
- `POST https://mcp.orchardhouse.cc/mcp` with no headers is rejected by Access.
- The same request with only the `mcp-proxy-clients` service token is rejected by mcpproxy.
- The same request with the service token and a client credential reaches mcpproxy.
- `docker ps` shows no published host port for mcpproxy.
- A container attached only to `mcp_internal` has no Traefik route.
