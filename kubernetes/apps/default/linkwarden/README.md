# Linkwarden

## Configuration

Documentation at [Linkwarden authentik](https://docs.linkwarden.app/self-hosting/sso-oauth#authentik)

Ensure the issuer is missing the trailing `/`.

Create a `NEXTAUTH_SECRET` with `openssl rand -base64 32`.

Setup the [Authentik device code flow](https://github.com/goauthentik/authentik/issues/5133).
