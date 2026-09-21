# OAuth callback restriction — #678

## Root cause and correction

XSUAA redirects to ARC-1's `/oauth/callback`; ARC-1 then redirects to the MCP client carried in
signed state. Shared CF/BAS wildcards trusted unrelated tenants at both boundaries. Narrowing
only `xs-security.json` leaves ARC-1's manual-client policy open.

Supply fixed supported client patterns to the existing provider and register exact deployment
callbacks, including the optional AppRouter. Keep the existing matcher, signed-state checks and
exact DCR binding. The UI helper preserves operator routes and OAuth restrictions when adding
its callback. Operator `.mtaext` files are gitignored and the previous helper preserved arbitrary
routing parameters; absence from tracked examples does not establish that no operator uses them.

## Evidence — 2026-09-21

- Removing only `redirectUriPatterns` makes four production-route tests fail: foreign CF/BAS
  authorization and replayed signed callback state become accepted. Restoring it passes.
- A UI-enabled MTAR with isolated names created XSUAA, Destination and Connectivity through
  MultiApps 3.11.1 in `us10-001` / `abap-dev`. Application deployment stopped at the CF route quota.
  Against its XSUAA service, `prompt=none` accepted exact backend callbacks and AppRouter
  `/login/callback` (302 `login_required`); foreign CF/BAS hosts and path suffixes received 400
  without a redirect. No authorization code or user token was issued.
- Separate disposable XSUAA instances accepted the shipped localhost descriptor. Localhost
  callback/logout paths with varying ports returned 302 `login_required`; a suffix, `127.0.0.1`
  or an HTTPS host returned 400. With exact deployed callbacks, both paths were accepted and
  foreign hosts/path suffixes refused. Tested entries match exactly, without an implicit path
  wildcard; public-URL prefixes therefore require explicit registration.
- Service-parameter readback was unsupported. Grants/lifetimes have descriptor/build coverage,
  not live readback. Temporary resources and keys were removed; existing apps/services unchanged.
- Route generation tests cover custom hosts, routes, plural hosts/domains, path prefixes,
  no-hostname and no-route, plus default output, OAuth restriction preservation and idempotency.
- SAP's [descriptor reference](https://help.sap.com/docs/btp/sap-business-technology-platform/application-security-descriptor-configuration-syntax)
  documents redirect allowlists; its [MTA reference](https://help.sap.com/docs/SAP_HANA_PLATFORM/4505d0bdaf4948449b7f7379d24d0f0d/33548a721e6548688605049792d55295.html)
  describes deployment substitution. Live requests establish the effective callback policy.

Deployed ARC-1/AppRouter login, token exchange and installed IDE sessions remain unverified
because of route quota. No completed security-plugin scan is claimed. R20's critical impact is
conditional on code exchangeability; these callback checks did not establish token compromise.

## Merge interactions

- `xs-security.json` conflicts with #813: keep #678's exact callback list. Its exact-list test
  carries #812 and subsumes #813's scheme-only describe. `mta-descriptor.test.ts` auto-merges
  and keeps both; deliberately delete `shipped xs-security.json redirect schemes (#812)`.
- `git merge-tree` also finds `docs_page/xsuaa-setup.md` conflicts. Combine #678's deployment,
  public-prefix and upgrade instructions with #813's HTTP(S)-only warning and troubleshooting.
  Describe the client gate as ARC-1's runtime policy; taking either complete file loses guidance.
- If included in the pending release, add a row to #813's release-note section and link the
  upgrade table and replace the `Unreleased` heading in `updating.md` with the actual version.
  #811 regenerated as 1.4.0 after #829; recheck its version before reconciliation.

Roadmap: no impact; SEC-15/SEC-16 and broader provider consent remain separate work.
