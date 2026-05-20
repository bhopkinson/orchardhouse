# Edge Auth With A Personal Microsoft Account

The edge stack uses `oauth2-proxy` with its `entra-id` provider. The provider name is still `entra-id`, but this stack is configured for personal Microsoft accounts only by using Microsoft's consumer tenant issuer:

```text
https://login.microsoftonline.com/9188040d-6c67-4c5b-b112-36a304b66dad/v2.0
```

This avoids the broader `common` authority and avoids disabling OIDC issuer verification.

## Microsoft App Registration

Create an app registration with these settings:

```text
Name: orchardhouse-oauth2-proxy
Supported account types: Personal Microsoft accounts only
Platform: Web
Redirect URI: https://auth.<your-domain>/oauth2/callback
```

Then create a client secret and use these values in Portainer's stack environment:

```text
OAUTH2_PROXY_CLIENT_ID=<application-client-id>
OAUTH2_PROXY_CLIENT_SECRET=<client-secret-value>
```

Generate the cookie secret on the Ubuntu host:

```bash
openssl rand -base64 32
```

Use the generated value as:

```text
OAUTH2_PROXY_COOKIE_SECRET=<generated-secret>
```

## Allow List

Update `portainer/stacks/edge/allowed-emails.txt` with the exact personal Microsoft email address that should be allowed to sign in.

The stack also sets:

```text
OAUTH2_PROXY_SCOPE=openid profile email
OAUTH2_PROXY_EMAIL_DOMAINS=*
OAUTH2_PROXY_AUTHENTICATED_EMAILS_FILE=/etc/oauth2-proxy/allowed-emails.txt
```

`email_domains=*` permits Microsoft to authenticate the account, and `authenticated_emails_file` restricts access back down to the email addresses in the allow-list file.
