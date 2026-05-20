# Edge Auth With Cloudflare Access And Microsoft Entra ID

The edge stack does not run a local authentication service. Authentication is enforced by Cloudflare Access before requests reach Traefik.

## What The Stack Does

- `cloudflared` connects the Ubuntu host to your existing Cloudflare Tunnel.
- `traefik` routes hostnames to containers on the internal Docker network.
- `whoami` is the first test application.

Cloudflare Access is responsible for:

- redirecting the user to Microsoft Entra ID,
- validating the user identity,
- enforcing the Access policy for the hostname,
- returning the user to the protected application.

## Cloudflare Access Setup

In Cloudflare Zero Trust:

1. Go to `Settings` -> `Authentication`.
2. Add `Microsoft Entra ID` as a login method.
3. Test the login method until Cloudflare reports that it works.

Then create Access applications for the private hostnames:

1. Go to `Access` -> `Applications`.
2. Create a `Self-hosted` application for `whoami.<your-domain>`.
3. Create another `Self-hosted` application for `traefik.<your-domain>`.
4. Attach an `Allow` policy for your user or group.
5. If Microsoft Entra ID is the only login method you want for these apps, turn on instant authentication.

Cloudflare's documentation recommends creating the Access application before publishing the tunnel route so the hostname is not briefly exposed without an Access policy.

## Tunnel Routing

Add these public hostnames to the existing Cloudflare Tunnel:

```text
whoami.<your-domain>   -> http://traefik:80
traefik.<your-domain>  -> http://traefik:80
```

Because `cloudflared` and `traefik` share the `orchard_proxy` Docker network, `cloudflared` can route to the `traefik` service name directly.

## Optional Hardening

Cloudflare recommends validating the Access token at the origin to protect against bypass routes. One supported option is enabling `Protect with Access` in the tunnel settings for the application hostname.

Start without it if you want the simplest first deployment, then add it once the basic flow is working.

## Future Website Hosting

Keep private infrastructure and public websites separate:

- `whoami.<your-domain>` and `traefik.<your-domain>` stay behind Access.
- future public websites should use different hostnames and should not have an Access application unless you want them private.
