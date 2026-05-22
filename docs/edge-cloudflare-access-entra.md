# Edge Auth With Cloudflare Access And Microsoft OIDC

This is the current working edge setup for the homelab.

The Docker stack does not run a local authentication service. Authentication is handled by Cloudflare Access before requests reach Traefik.

## Current Working State

- `cloudflared` connects the Ubuntu Docker host to the existing Cloudflare Tunnel.
- `traefik` receives tunnel traffic and routes by hostname.
- `whoami` is the first private test application.
- Cloudflare Tunnel dashboard routes are empty.
- Tunnel ingress config lives inline in `portainer/stacks/edge/compose.yml`.
- Hostnames use a single subdomain level, such as `whoami.<your-domain>`.
- Cloudflare Access protects private hostnames.
- Microsoft personal account login works through Cloudflare's generic `OpenID Connect` provider using Microsoft's `consumers` endpoints.

## Hostnames

The private test hostnames are:

```text
whoami.<your-domain>
traefik.<your-domain>
```

For this homelab:

```text
whoami.orchardhouse.cc
traefik.orchardhouse.cc
```

The earlier `whoami.apps.<your-domain>` pattern is not used because Cloudflare Universal SSL does not cover nested wildcard hostnames such as `*.apps.orchardhouse.cc` without a paid advanced certificate.

## Cloudflare DNS

Create one wildcard DNS record:

```text
Type: CNAME
Name: *
Target: <tunnel-id>.cfargotunnel.com
Proxy status: Proxied
```

This covers one subdomain level, for example:

```text
whoami.orchardhouse.cc
traefik.orchardhouse.cc
ha.orchardhouse.cc
www.orchardhouse.cc
```

It does not cover the bare apex domain:

```text
orchardhouse.cc
```

If the apex domain is needed later, create a separate DNS record or Cloudflare Tunnel route for it.

## Microsoft App Registration

Create a Microsoft app registration for Cloudflare Access.

In the Microsoft Entra admin center:

1. Go to `Identity` -> `Applications` -> `App registrations`.
2. Select `New registration`.
3. Set `Name` to `Cloudflare Access Homelab`.
4. Under `Supported account types`, select:

```text
Accounts in any organizational directory and personal Microsoft accounts
```

5. Under `Redirect URI`, select `Web`.
6. Enter this redirect URI, replacing `<team-name>` with the Cloudflare Zero Trust team name:

```text
https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback
```

7. Select `Register`.
8. Copy the `Application (client) ID`.

Create a client secret:

1. Go to `Certificates & secrets`.
2. Select `New client secret`.
3. Give it a description such as `Cloudflare Access`.
4. Choose an expiry you are comfortable rotating.
5. Copy the secret `Value` immediately.

Use the secret `Value`, not the `Secret ID`.

If checking the app manifest, this value should be present:

```json
"signInAudience": "AzureADandPersonalMicrosoftAccount"
```

If an existing app registration cannot be changed to that value, create a new app registration with the correct supported account type from the start.

## Cloudflare Identity Provider

Use Cloudflare's generic `OpenID Connect` provider. Do not use the built-in `Microsoft Entra ID` provider for personal Microsoft accounts.

The built-in Entra provider is tenant-based and can produce this error for personal accounts:

```text
You can't sign in here with a personal account. Use your work or school account instead.
```

In Cloudflare Zero Trust:

1. Go to `Settings` -> `Team name and domain`.
2. Confirm the team name matches the redirect URI in the Microsoft app registration.
3. Go to `Integrations` -> `Identity providers`.
4. Select `Add new identity provider`.
5. Choose `OpenID Connect`.
6. Name it `Microsoft Personal OIDC`.
7. In `App ID`, enter the Microsoft `Application (client) ID`.
8. In `Client Secret`, enter the Microsoft client secret `Value`.
9. Use these OIDC URLs:

```text
Auth URL:        https://login.microsoftonline.com/consumers/oauth2/v2.0/authorize
Token URL:       https://login.microsoftonline.com/consumers/oauth2/v2.0/token
Certificate URL: https://login.microsoftonline.com/consumers/discovery/v2.0/keys
```

10. Use these scopes:

```text
openid
email
profile
```

11. Set the email claim name to:

```text
email
```

12. Enable PKCE if Cloudflare offers the option.
13. Save the identity provider.
14. Use Cloudflare's `Test` button and sign in with the personal Microsoft account.

If Cloudflare does not show an email address during the provider test, try this email claim instead:

```text
preferred_username
```

Then use the exact email or username Cloudflare reports in the Access policy.

## Cloudflare Access Applications

Use a wildcard Access application while this homelab is private-first.

Create one self-hosted Access application:

1. Go to `Access` -> `Applications`.
2. Create a `Self-hosted` application.
3. Set the hostname to:

```text
*.orchardhouse.cc
```

4. Add an `Allow` policy for the personal Microsoft account.
5. Set the application to use only `Microsoft Personal OIDC`.
6. Turn on instant authentication if this is the only login method.

This means new one-level hostnames are private by default:

```text
whoami.orchardhouse.cc
traefik.orchardhouse.cc
portainer.orchardhouse.cc
ha.orchardhouse.cc
```

That is convenient while rebuilding the homelab because new internal services do not need a separate Access application before they are protected.

Use more specific Access applications for exceptions, such as machine webhooks or future public websites. Cloudflare path-specific applications can override the broader wildcard application when the hostname and path match.

Cloudflare recommends creating the Access application before publishing the tunnel route so the hostname is not briefly exposed without an Access policy.

## Tunnel Routing

The Cloudflare Tunnel dashboard routes should be empty for this stack.

The route lives inline in `portainer/stacks/edge/compose.yml`:

```yaml
configs:
  cloudflared_config:
    content: |
      ingress:
        - service: http://traefik:80
```

Because `cloudflared` and `traefik` share the `orchard_proxy` Docker network, `cloudflared` can route to the `traefik` service name directly. Traefik then chooses the final container from the hostname.

## Portainer Environment Variables

Set these environment variables on the Portainer stack:

```text
DOMAIN=<your-domain>
CLOUDFLARE_TUNNEL_ID=<your-tunnel-uuid>
```

For this homelab:

```text
DOMAIN=orchardhouse.cc
CLOUDFLARE_TUNNEL_ID=<your-tunnel-uuid>
```

The tunnel ID is the UUID of the tunnel, not the dashboard token.

## Tunnel Credentials

The Ubuntu Docker host must have the tunnel credentials JSON file at:

```text
/srv/orchardhouse/cloudflared/credentials.json
```

Do not commit this file to Git.

The `cloudflared` container must be able to read the file. On the Ubuntu host, use:

```bash
sudo install -d -m 755 /srv/orchardhouse/cloudflared
sudo chown root:root /srv/orchardhouse/cloudflared/credentials.json
sudo chmod 644 /srv/orchardhouse/cloudflared/credentials.json
sudo chmod 755 /srv /srv/orchardhouse /srv/orchardhouse/cloudflared
```

To check the permissions:

```bash
ls -ld /srv /srv/orchardhouse /srv/orchardhouse/cloudflared
ls -l /srv/orchardhouse/cloudflared/credentials.json
```

The directories should be readable and executable by others, and the credentials file should be readable by the container.

## Private And Public Hostnames

The same tunnel and Traefik instance can carry both private and public sites.

Cloudflare Access decides which hostnames require login.

The current setup uses one wildcard Access application for private hostnames:

```text
*.orchardhouse.cc
```

This makes new apps private by default.

Future public hostnames need an explicit exception, for example a more specific Access application with a `Bypass` policy:

```text
www.orchardhouse.cc
blog.orchardhouse.cc
```

When a public website is added later, it will also get its own Traefik router label for that hostname.

Portainer has extra notes because its frontend uses the wildcard human-login policy, while its GitOps webhook uses a more specific service-token policy. See `docs/portainer-tunnel-gitops.md`.

## Troubleshooting

If the Cloudflare identity provider test works but an Access application still rejects personal Microsoft accounts, check the Microsoft login URL shown in the browser.

The URL should contain:

```text
login.microsoftonline.com/consumers/
```

It should not contain:

```text
login.microsoftonline.com/common/
```

Also compare the URL's `client_id` value with the Microsoft app registration's `Application (client) ID`.

If the `client_id` is different, the Access application is still using an old identity provider or another matching Access application is taking priority.

Check:

1. The wildcard `*.orchardhouse.cc` Access app uses only `Microsoft Personal OIDC`.
2. The old tenant-based `Microsoft Entra ID` provider is disabled or deleted.
3. The Microsoft app registration has `signInAudience` set to `AzureADandPersonalMicrosoftAccount`.
4. No more specific Access application is accidentally matching the same hostname or path with the wrong provider.

## Optional Hardening

Cloudflare recommends validating the Access token at the origin to protect against bypass routes. One supported option is enabling `Protect with Access` in the tunnel settings for the application hostname.

Start without it for the simplest first deployment, then add it once the basic flow is stable.
