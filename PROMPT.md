# My Library Account: specification for AI-assisted builds

A web app that shows an Alma patron their active requests, current loans, and fees after
signing in with the institution's identity provider. This file is the spec: give it to a
coding assistant in an empty repository and ask it to build the app. Institution-specific
values stay in configuration, never in the spec.

## 1. Purpose

- Patrons see their own Alma account data without logging in to Alma or Primo.
- The library's Alma API key stays on the server. The browser never sees it.
- One deployable artifact: a Spring Boot jar that serves the React frontend.

## 2. Stack and constraints

- Backend: Spring Boot 3.x, Java 21, Maven. Spring Security for login.
- Frontend: React 19 with Material UI, built by Vite, packaged into the jar by the
  frontend-maven-plugin. Package manager: pnpm (change to npm if your team prefers).
- The backend proxies all Alma calls. The frontend only talks to `/api/**` on the same origin.
- Session-cookie authentication between browser and backend. No tokens in the browser.

## 3. Authentication

Default variant: Amazon Cognito. See section 9 for other identity providers.

- OpenID Connect, authorization code flow, via Spring Security's oauth2 client with an
  issuer URI so endpoints are discovered.
- Request only the `openid` scope.
- Use `sub` as Spring's `user-name-attribute`. It must exist in both the ID token and the
  userinfo response, and `sub` always does.
- The Alma user identifier is read from an ID-token claim whose name is configurable
  (default `custom:almaUser`). If the claim is missing, API calls return 400 with a clear
  message. Never fall back to another value such as the username.
- Sign-out clears the local session, then redirects to the Cognito Hosted UI logout endpoint,
  which needs the Hosted UI domain and a registered sign-out URL.
- Unauthenticated browser navigation redirects to login. Unauthenticated `/api/**` calls
  return 401 and the frontend reloads the page to trigger login.

## 4. Alma integration

- Alma Users API with a server-side API key that has read access to users, requests, loans,
  and fees.
- Accept the regional gateway URL with or without the `/almaws/v1` path and with or without a
  trailing slash.
- Endpoints: `/users/{id}/requests?status=active`, `/users/{id}/loans`,
  `/users/{id}/fees?status=ACTIVE`. Follow pagination with limit 100 and concatenate pages.
- Pass Alma's JSON through to the frontend unchanged so columns can use any field.
- Map Alma's "user not found" error (code 401861) to HTTP 404. Map other failures to 502.
  Log the request path, status, and the first part of the response body on every failure.

## 5. Functional requirements

Each item is testable.

1. After login, the page shows three tables: Active requests, Loans, Fees.
2. Requests table: title, author, type, status, pickup location, placed date, expiry date.
3. Loans table: title, author, call number, library, location, checkout date, due date,
   barcode. Overdue due dates are highlighted.
4. Fees table: type, title, status, created date, owner, original amount, balance, comment,
   plus a total due line using Alma's currency.
5. Each table shows a row count, a loading indicator, an empty-state message, and an error
   message if its API call fails. One table failing does not blank the others.
6. The app bar shows the signed-in user's name or ID, a refresh button, and a sign-out button.
7. `/api/me` returns the user ID the app will send to Alma, the claim it was read from, the
   names (not values) of all claims received, and the CSRF token needed for sign-out.
8. Dates render in the browser's locale. Amounts render as currency.

## 6. Configuration

Environment variables, with defaults in `application.yml`, documented in README and
`.env.example`:

| Variable                  | Purpose                                                   |
|---------------------------|-----------------------------------------------------------|
| `ALMA_BASE_URL`           | Regional Alma gateway, e.g. `https://api-na.hosted.exlibrisgroup.com` |
| `ALMA_API_KEY`            | Read-only Users API key                                   |
| `ALMA_USER_ID_CLAIM`      | ID-token claim holding the Alma identifier                |
| `COGNITO_ISSUER_URI`      | `https://cognito-idp.<region>.amazonaws.com/<pool-id>`    |
| `COGNITO_CLIENT_ID`       | App client ID                                             |
| `COGNITO_CLIENT_SECRET`   | App client secret, empty for a public client              |
| `COGNITO_DOMAIN`          | Hosted UI domain, used for logout                         |
| `COGNITO_LOGOUT_REDIRECT` | Where Cognito returns the browser after sign-out          |

Identity provider settings to document: callback URL
`<app>/login/oauth2/code/cognito`, sign-out URL, allowed scopes, and read permission on the
attribute that carries the Alma identifier.

## 7. Acceptance criteria and tests

- Alma client unit tests with a mock server: pagination across two pages, the `status`
  filters, fees totals, empty lists returned as empty arrays, user-not-found mapped to 404,
  base URL normalization.
- Security integration tests with a fake OIDC principal: browser navigation redirects to
  login; `/api/**` without a session returns 401; the configured claim is passed to the Alma
  client; a missing claim returns 400 and does not call Alma; static assets and `/api` typos
  are never answered with the SPA shell.
- `mvn clean package` builds the frontend, runs the tests, and produces a runnable jar that
  serves the app at `/`.

## 8. Out of scope

- Renewing loans, cancelling requests, or paying fees. Read-only for now.
- Staff views or looking up other patrons.
- Mobile app packaging.

## 9. Other identity providers

Replace section 3 with the matching variant. Everything else is unchanged.

**Any OIDC provider** (Okta, Microsoft Entra ID, Keycloak, Shibboleth OIDC, Auth0): keep the
standard oauth2 client registration with an issuer URI, request `openid profile email`, and
set the Alma claim to the provider's username claim, typically `preferred_username`, `eppn`,
or `uid`. These providers publish `end_session_endpoint`, so use Spring's
`OidcClientInitiatedLogoutSuccessHandler` instead of a custom logout handler.

**Social logins** (Google, Microsoft personal, Apple, GitHub): the token has no Alma ID, only a
verified email. Require `email_verified`, look the patron up in Alma by email using the
identifier type or a `/users?q=email~` search, and require exactly one match. Optionally
restrict to an institutional domain (Google `hd` claim). Treat email as a lookup key, not as
proof of identity for anything beyond reading the patron's own account.

**SAML** (Shibboleth, ADFS, Okta SAML, Azure AD SAML): use Spring Security `saml2Login`
configured from IdP metadata, with the Alma identifier read from a configurable SAML
attribute (for `uid`, `urn:oid:0.9.2342.19200300.100.1.1`). The controller principal becomes
`Saml2AuthenticatedPrincipal` and the lookup becomes `getFirstAttribute`.

**Cognito brokering any of the above**: the main spec applies unchanged. Federated usernames
take the form `<provider>_<id>`; map whichever form Alma stores into the custom attribute.
Read the Alma claim from the ID token; in our testing it was absent from userinfo.

## 10. Known pitfalls

All configuration issues, none of them code:

- Requesting `profile` or `email` scopes the app client doesn't allow causes `invalid_scope`.
- A `user-name-attribute` that is only in the ID token fails at the userinfo step with
  "Attribute value for '...' cannot be null". Use `sub`.
- `ALMA_BASE_URL` ending in `/almaws/v1/` doubled the path and produced a gateway 404 that
  looked like a user lookup failure. The client must normalize the URL.
- An SPA fallback mapping of `/{path}/**` captures `/assets/*.js` and serves HTML, which
  Firefox reports as `NS_ERROR_CORRUPTED_CONTENT`. Spell out dot-free path segments instead.
