# Portainer Tunnel And GitOps Webhooks

This exposes Portainer through the existing Cloudflare Tunnel and keeps GitOps webhooks working without opening Portainer directly to the internet.

## Current Design

There are two Portainer hostnames:

```text
portainer.orchardhouse.cc
portainer-webhook.orchardhouse.cc
```

Use them differently:

- `portainer.orchardhouse.cc` is for humans and is protected by the wildcard Cloudflare Access login application for `*.orchardhouse.cc`.
- `portainer-webhook.orchardhouse.cc` is for GitHub Actions and has a more specific path-based Cloudflare Access application protected by a service token.

The webhook hostname only has a Traefik route for:

```text
/api/stacks/webhooks/
```

Other paths on that hostname should not route to Portainer.

## How Traffic Flows

For the Portainer frontend:

```text
Browser
Cloudflare Access
Cloudflare Tunnel
cloudflared
Traefik
Portainer
```

For GitOps webhooks:

```text
GitHub Actions
Cloudflare Access service token
Cloudflare Tunnel
cloudflared
Traefik webhook path router
Portainer stack webhook
```

## Repo Changes That Matter

The Portainer bootstrap compose file now has Traefik labels:

```text
bootstrap/portainer/compose.yml
```

The edge stack now attaches Traefik to Portainer's Docker network:

```text
portainer/stacks/edge/compose.yml
```

The edge GitHub Actions workflow now sends Cloudflare Access service-token headers:

```text
.github/workflows/portainer-stacks-edge.yml
```

## Apply The Portainer Bootstrap Change

Portainer manages the other stacks, but Portainer itself is still bootstrapped locally with Docker Compose.

On the Ubuntu Docker host, pull the latest repo changes, then from the repo directory run:

```bash
cd bootstrap/portainer
cp .env.example .env
nano .env
```

Set:

```text
DOMAIN=orchardhouse.cc
```

Then redeploy Portainer:

```bash
docker compose pull
docker compose up -d
```

The Portainer `--trusted-origins` value must be a full URL, including the scheme:

```text
https://portainer.orchardhouse.cc/
```

The compose file binds Portainer's raw HTTPS port to localhost only:

```text
127.0.0.1:9443:9443
```

That means Portainer should be reached through the tunnel at `https://portainer.orchardhouse.cc`, not directly from another machine on the LAN using port `9443`.

## Redeploy The Edge Stack

After Portainer has been redeployed locally, redeploy the `edge` stack in Portainer.

This attaches Traefik to the existing `portainer_network` network so Traefik can route to the Portainer container.

## Cloudflare Access For The Frontend

The Portainer UI is covered by the existing wildcard Access application:

```text
Hostname: *.orchardhouse.cc
Policy: Allow your personal Microsoft account
Identity provider: Microsoft Personal OIDC
```

You do not need a separate Access application for `portainer.orchardhouse.cc` while the wildcard application is in place.

Turn on instant authentication on the wildcard application if `Microsoft Personal OIDC` is the only login method.

## Portainer Login And SSO

Cloudflare Access protects the Portainer frontend, but Portainer still has its own application session.

Portainer does not use Cloudflare Access headers as a native login method. The supported approach is to configure Portainer's own OAuth login against Microsoft as well.

The result is close to SSO:

- Cloudflare Access checks identity before Portainer loads.
- Portainer uses OAuth for its own session.
- Because the Microsoft session already exists, Portainer should not usually ask for Microsoft credentials again.

You may still see a Portainer OAuth login button because Portainer has to create its own session.

### Microsoft App Registration For Portainer

Create a separate Microsoft app registration for Portainer OAuth.

In the Microsoft Entra admin center:

1. Go to `Identity` -> `Applications` -> `App registrations`.
2. Select `New registration`.
3. Set `Name` to `Portainer Homelab OAuth`.
4. Under `Supported account types`, select:

```text
Accounts in any organizational directory and personal Microsoft accounts
```

5. Under `Redirect URI`, select `Web`.
6. Add:

```text
https://portainer.orchardhouse.cc/
```

7. Register the app.
8. Copy the `Application (client) ID`.
9. Create a client secret and copy the secret `Value`.

### Portainer OAuth Settings

In Portainer:

1. Log in with the local `admin` user first.
2. Go to `Settings` -> `Authentication`.
3. Select `OAuth`.
4. Choose the custom OAuth provider option.
5. Enable `Use SSO`.
6. Leave `Hide internal authentication prompt` off until OAuth login has been tested.
7. Enable automatic user provisioning if you want Portainer to create your OAuth user automatically.

Use these values:

```text
Client ID:        <Microsoft Application (client) ID>
Client secret:    <Microsoft client secret Value>
Authorization URL: https://login.microsoftonline.com/consumers/oauth2/v2.0/authorize
Access token URL:  https://login.microsoftonline.com/consumers/oauth2/v2.0/token
Resource URL:      https://graph.microsoft.com/oidc/userinfo
Redirect URL:      https://portainer.orchardhouse.cc/
Logout URL:        https://login.microsoftonline.com/consumers/oauth2/v2.0/logout
User identifier:   email
Scopes:            openid email profile
Auth style:        Auto, if available
```

If `email` does not work as the user identifier, try:

```text
preferred_username
```

After the OAuth user can log in, make sure the OAuth user has the Portainer permissions you expect. For a single-user homelab, that usually means admin access.

Only hide the internal authentication prompt after OAuth works. Keep the initial local admin account as a recovery path.

## Cloudflare Access For GitHub Webhooks

Create a Cloudflare Access service token:

1. Go to `Access` -> `Service Auth` -> `Service Tokens`.
2. Create a token named `github-actions-portainer-webhook`.
3. Copy the `Client ID`.
4. Copy the `Client Secret`.

Create a more specific self-hosted Access application for the webhook path:

```text
Hostname: portainer-webhook.orchardhouse.cc
Path: /api/stacks/webhooks/*
Policy action: Service Auth
Include: github-actions-portainer-webhook
```

Do not create a normal Allow policy for this webhook application.

This path-specific service-token application is the exception to the broader wildcard login application. Browser users still hit the wildcard Microsoft login for normal homelab hostnames, while GitHub Actions can call only the Portainer webhook path using service-token headers.

## Portainer Stack Webhook

In Portainer:

1. Go to `Stacks`.
2. Open the `edge` stack.
3. Open the editor/settings area where webhooks are configured.
4. Enable `Create a stack webhook`.
5. Copy the generated webhook URL.

Portainer may show a URL using its internal or local hostname. Keep the path and token, but replace the scheme and hostname with:

```text
https://portainer-webhook.orchardhouse.cc
```

The final URL should look like:

```text
https://portainer-webhook.orchardhouse.cc/api/stacks/webhooks/<webhook-token>
```

## GitHub Secrets

In GitHub, go to `Settings` -> `Secrets and variables` -> `Actions` and add:

```text
PORTAINER_EDGE_WEBHOOK=https://portainer-webhook.orchardhouse.cc/api/stacks/webhooks/<webhook-token>
CF_ACCESS_CLIENT_ID=<Cloudflare service token client ID>
CF_ACCESS_CLIENT_SECRET=<Cloudflare service token client secret>
```

The `portainer-stacks-edge` workflow validates the Compose file, then calls the webhook with:

```text
CF-Access-Client-Id
CF-Access-Client-Secret
```

## Test The Webhook

After adding the GitHub secrets, make a small commit that changes something under:

```text
portainer/stacks/edge/
```

Push it to `main`, then check:

1. The GitHub Actions workflow succeeds.
2. Portainer shows the `edge` stack redeploying.
3. `whoami.orchardhouse.cc` still works.
4. `portainer.orchardhouse.cc` loads through Cloudflare Access.

## Why Not Put The Webhook Under The Same Hostname?

The frontend and webhook have different security needs.

The frontend is for browser users and should require Microsoft login.

The webhook is for GitHub Actions and should not need browser login. A Cloudflare Access service token is a better match because GitHub Actions can send fixed headers with the request.

Keeping the webhook on `portainer-webhook.orchardhouse.cc` also makes the Traefik route easy to restrict to only the Portainer webhook path.

## Future Public Sites

The wildcard Access app means new hostnames are private by default. That is useful for internal tools, but public websites need an explicit exception.

For a future public site such as:

```text
www.orchardhouse.cc
```

create a more specific Access application for that hostname with a `Bypass` policy, or replace the wildcard Access approach with exact private Access applications once public hosting becomes a bigger part of the setup.

Using `Bypass` keeps the private-by-default model while letting selected public hostnames through without login.
