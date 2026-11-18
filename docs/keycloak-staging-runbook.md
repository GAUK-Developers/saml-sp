# Runbook: Testing Staging Keycloak with saml-sp

This runbook starts the local `saml-sp` test app on `http://localhost:7070` and uses it to check that staging Keycloak works as a SAML 2.0 Identity Provider (IdP), through the `urn:example:sp` client.

It covers SP-initiated login, IdP-initiated login, signature and NameID checks, and single logout (SLO). It does not test Keycloak's OIDC/OAuth clients.

Staging Keycloak is `https://identity-staging.groupe-atlantic.co.uk`, realm `ideal-identity-broker`.

## 1. Prerequisites

- [ ] Node.js 10 or later on your machine (`node -v`)
- [ ] This repo checked out, with dependencies installed (`npm install` in the repo root)
- [ ] Port 7070 free on localhost (`lsof -i :7070` prints nothing)
- [ ] Admin console access to that realm, so you can view and edit the `urn:example:sp` client
- [ ] A test user in the realm with a password and an email address set
- [ ] Your browser can reach staging Keycloak (turn on the VPN if staging needs it)

## 2. Check the Keycloak client

The staging `urn:example:sp` client is close to right, but **Master SAML Processing URL is set to `/saml/consume`, and saml-sp has no route at that path**. saml-sp receives assertions at `/saml/sso`.

- SP-initiated login may still work. The SP sends its ACS URL in the AuthnRequest, and Keycloak uses it because it matches the valid redirect URIs.
- IdP-initiated login will post to `/saml/consume` and fail with a 404.

In the admin console, go to **Clients → urn:example:sp** and check each value:

| Tab / section | Field | Current | Required |
| --- | --- | --- | --- |
| Settings → General | Client ID | `urn:example:sp` | `urn:example:sp` (the same as saml-sp's default `--audience`) |
| Settings → Access | Root URL / Home URL | `http://localhost:7070` | No change |
| Settings → Access | Valid redirect URIs | `http://localhost:7070/*` | No change |
| Settings → Access | Valid post logout redirect URIs | `http://localhost:7070/*` | No change |
| Settings → Access | IDP-Initiated SSO URL name | `local-test-sp` | No change |
| Settings → Access | Master SAML Processing URL | `http://localhost:7070/saml/consume` | **`http://localhost:7070/saml/sso`** |
| Settings → SAML capabilities | Name ID format | Not checked | `email` |
| Settings → SAML capabilities | Force name ID format | Not checked | On |
| Settings → Signature and Encryption | Sign documents / Sign assertions | Not checked | On (either one is enough) |
| Keys | Client signature required | Not checked | On, with saml-sp's `config/sp-cert.pem` imported. The realm metadata has `WantAuthnRequestsSigned="true"`, so expect Keycloak to check signatures. |
| Advanced → Fine grain SAML endpoint configuration | Assertion Consumer Service POST Binding URL | Not checked | `http://localhost:7070/saml/sso`, or empty |
| Advanced → Fine grain SAML endpoint configuration | Logout Service Redirect Binding URL | Not checked | `http://localhost:7070/saml/slo` |

"Not checked" means the field wasn't in the screenshot of the Settings tab.

If you can't change the staging client, start saml-sp with `--acsUrls /saml/sso /saml/consume` (see step 3). saml-sp then listens on both paths.

To import the SP certificate: go to **Keys → Import key**, choose the archive format *Certificate PEM*, and upload `config/sp-cert.pem` from this repo.

## 3. Start saml-sp

From the repo root:

```bash
node bin/server.js \
  --idpMetaUrl https://identity-staging.groupe-atlantic.co.uk/realms/ideal-identity-broker/protocol/saml/descriptor
```

This instance serves realm metadata at `/realms/...` (the `/auth/realms/...` path returns 404).

The SP reads the SSO URL, SLO URL and signing certificate from the realm metadata. Defaults already match the staging client: `--audience urn:example:sp`, port `7070`, ACS `/saml/sso`, signed AuthnRequests (RSA-SHA256).

Optional flags:

| Flag | When to use it |
| --- | --- |
| `--acsUrls /saml/sso /saml/consume` | Master SAML Processing URL is still `/saml/consume` and you can't change it |
| `--validateNameIDFormat false` | Keycloak sends a NameID format other than `email` |
| `--signed false` | Client signature required is On and the SP cert isn't imported |
| `--requestAuthnContext false` | Keycloak rejects the requested AuthnContext (`PasswordProtectedTransport`) |

Check that it started:

- The terminal prints the IdP settings it loaded from the metadata, with no errors.
- `http://localhost:7070/metadata` returns the SP metadata XML.
- `http://localhost:7070/settings` shows the IdP SSO URL pointing at `identity-staging.groupe-atlantic.co.uk`.

Leave the terminal open. saml-sp logs every AuthnRequest and SAMLResponse there, which is the main debugging aid.

## 4. Test cases

Use a private browser window for each test, so old Keycloak sessions don't hide problems.

### 4.1 SP-initiated login

1. Open `http://localhost:7070/login`.
2. The browser redirects to the staging Keycloak login page for `ideal-identity-broker`.
3. Sign in as the test user.
4. **Expected:** the browser lands on `http://localhost:7070/profile`, showing the user's NameID (their email), the session index and the attributes.
5. In the terminal, check that `Received SSO Response on ACS URL http://localhost:7070/saml/sso` is logged with no validation error.

### 4.2 IdP-initiated login

1. Open `https://identity-staging.groupe-atlantic.co.uk/realms/ideal-identity-broker/protocol/saml/clients/local-test-sp`.
2. Sign in as the test user (or you're signed in already, if a Keycloak session exists).
3. **Expected:** Keycloak posts to `http://localhost:7070/saml/sso` and the profile page appears.
4. If you get a 404 at `/saml/consume`, the Master SAML Processing URL fix from step 2 hasn't been applied.

### 4.3 Assertion contents

On the profile page after either login, check:

- [ ] **NameID** is the user's email, with format `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`
- [ ] **Issuer** is `https://identity-staging.groupe-atlantic.co.uk/realms/ideal-identity-broker`
- [ ] **Audience** is `urn:example:sp`
- [ ] The **attributes** you expect from the client's mappers (for example, roles or email) are present
- [ ] The raw SAMLResponse in the terminal has a `<ds:Signature>` on the response or the assertion

### 4.4 Logout (SLO)

1. While signed in, open `http://localhost:7070/logout`.
2. **Expected:** the SP sends a LogoutRequest to Keycloak, Keycloak ends its session, and the browser comes back to the saml-sp logout page.
3. Open `http://localhost:7070/login` again. **Expected:** Keycloak asks for the password again, which confirms the Keycloak session ended.
4. Optionally, check in **Sessions** (on the client or the user) in the admin console that the session is gone.

### 4.5 Negative check (optional)

1. Stop saml-sp (Ctrl+C) and start it again with `--audience urn:wrong:sp`.
2. Open `http://localhost:7070/login`.
3. **Expected:** Keycloak shows an error page saying the client isn't found or the request is invalid. That confirms Keycloak only issues assertions to registered clients.
4. Restart saml-sp with the normal command from step 3 afterwards.

## 5. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| saml-sp fails at startup while fetching metadata | Wrong host or realm, VPN off, or the old `/auth` path is needed | Open the metadata URL in a browser to check it |
| Keycloak: "Invalid requester" | Client signature required is On, but Keycloak doesn't have the SP cert | Import `config/sp-cert.pem` in **Keys**, or start the SP with `--signed false` |
| Keycloak: "Client not found" | `--audience` doesn't match the Client ID | Use `urn:example:sp` (the default) |
| Keycloak: "Invalid redirect uri" | The ACS URL isn't covered by the valid redirect URIs | Keep `http://localhost:7070/*` |
| 404 at `/saml/consume` | Master SAML Processing URL still points to `/saml/consume` | Change it to `/saml/sso`, or start the SP with `--acsUrls /saml/sso /saml/consume` |
| SP error about the NameID format | Keycloak sends `username` or `unspecified` | Set Name ID format to `email` in Keycloak, or start the SP with `--validateNameIDFormat false` |
| SP error about the signature or certificate | The realm's signing key was rotated after saml-sp started | Restart saml-sp to reload the metadata |
| SP error about the response being expired | Clock skew between your machine and Keycloak | Sync your system clock |
| Logout returns without ending the Keycloak session | No SLO URL in the metadata, or no logout URL on the client | Set the Logout Service Redirect Binding URL to `http://localhost:7070/saml/slo` |
