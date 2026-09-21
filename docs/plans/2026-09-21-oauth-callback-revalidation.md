# OAuth callback restriction revalidation (#678)

## Root cause and choice

ARC-1 proxies two distinct redirects. XSUAA sends its authorization response to
ARC-1's `/oauth/callback`. ARC-1 then sends it to the MCP client's callback carried
in signed state. The original shared-platform wildcards trusted unrelated CF/BAS
tenants at both boundaries. Narrowing only `xs-security.json` leaves ARC-1's
manual-client policy unchanged.

The PR's existing provider option is the smallest runtime correction: supply a
fixed list of supported manual client callbacks. Keep the dependency's existing
URL matcher, signed-state verification, and exact DCR registration binding.
Do not add another matcher, callback registry, or configuration setting.

The deployment changes are also necessary: register exact backend paths and the
optional AppRouter path. The existing UI extension helper must preserve operator
routes and OAuth restrictions when adding the UI callback. Requiring everyone to
hand-edit generated callbacks would break the documented deployment helper.

## Plan

1. Merge current main and review every changed file plus the provider enforcement
   points. Use the repository threat model for a bounded security diff review.
2. Reproduce the unsafe runtime behavior by removing only the provider policy
   option, then restore it and rerun the same production-route tests.
3. Validate the MTA combinations and build/deploy an isolated test copy with unique
   application/service/role-collection names. Preserve the callback expressions,
   shipped runtime, and merge behavior being tested. Exercise accepted and refused
   callbacks against the deployed backend and XSUAA; remove test resources.
4. Add or adjust only changes justified by a reproduced defect or a validation gap.
5. Run focused and full tests, static checks, strict docs, and package/deployment
   checks. Record exact live coverage and remaining login limitations.
6. Push additive review commits to #678. No roadmap impact after checking the
   inventory: this does not implement SEC-15, SEC-16, or broader provider consent.

## Evidence so far

On main merged at `60a439fa`, removing the explicit policy causes four of the
20 production-route OAuth tests to fail: foreign shared-platform authorization
and old signed callback state become accepted. Restoring it passes the focused
suite. This is an independent rerun against `@arc-mcp/xsuaa-auth` 1.1.0.

SAP's [descriptor reference](https://help.sap.com/docs/btp/sap-business-technology-platform/application-security-descriptor-configuration-syntax)
documents explicit redirect allowlists and cautions against broad wildcards.
Its [MTA descriptor reference](https://help.sap.com/docs/SAP_HANA_PLATFORM/4505d0bdaf4948449b7f7379d24d0f0d/33548a721e6548688605049792d55295.html)
explains deploy-time URL substitution; a static validation alone cannot establish
the final XSUAA broker payload.

## Results (2026-09-21)

- All 6,924 unit tests in 225 files passed, including 45 focused OAuth/MTA/UI
  tests. Typecheck, build, lint (two existing informational notices), policy and
  size/schema checks, strict docs, and all five MTA validation combinations passed.
- Built a complete UI-enabled MTAR from this revision in an isolated source copy.
  Test application, service, and role-collection names used a unique `pr678`
  prefix; the callback expressions and runtime were retained.
- Deployed through MultiApps 3.11.1 into the authorized us10-001 `abap-dev` test
  space. MTA resolved its URL references and successfully created XSUAA,
  Destination, and Connectivity. CF then refused application routes with
  `Routes quota exceeded for organization`; no application login was possible.
- Against that MTA-created XSUAA service, `prompt=none` accepted both exact
  backend paths and the exact optional AppRouter `/login/callback` with HTTP 302
  and `login_required`. Foreign CF/BAS hosts and a suffix on the callback path
  returned HTTP 400 without a redirect. No authorization code or user token was
  issued. The temporary test key was deleted.
- Service-parameter readback is unsupported by this broker. The accepted/rejected
  requests prove effective callback policy; they do not prove token lifetimes by
  live readback. Grants/lifetimes are covered by descriptor and build checks.
- The code review traced all changed files and both provider enforcement points.
  The security plugin's report finalization failed with
  `scan-manifest.json: expected a file inside the scan directory`; its semantic
  draft was subsequently saved, but no completed scan is claimed.

## Independent revalidation (second reviewer, 2026-09-21)

- Removing only `redirectUriPatterns` from the `createXsuaaOAuthProvider` call again fails exactly
  four of the production-route tests — the two shared-platform hosts at `/authorize`, and the two
  replayed signed states at `/oauth/callback`. Restoring it passes. `npm test`: 6,924 tests in
  225 files. Typecheck, build, lint (two pre-existing informational notices), `validate:policy`,
  `check:sizes`, all five `mbt validate` combinations and strict MkDocs pass.
- Live XSUAA on CF `us10-001` / `abap-dev`, disposable unbound instances, deleted with their keys
  afterwards:
  - This PR's shipped `xs-security.json` (`http://localhost:*/oauth/{callback,logged-out}`) is
    accepted by the broker; `prompt=none` returns HTTP 302 `login_required` for
    `http://localhost:6274/oauth/callback` and `http://localhost:3000/oauth/logged-out`, and
    HTTP 400 for a path suffix, for `http://127.0.0.1:6274/oauth/callback` and for an https host.
  - With the two exact deployed callbacks registered (the shape `mta.yaml` produces), `prompt=none`
    returns 302 `login_required` for both, and 400 with no redirect for
    `…/oauth/callback/extra`, for a foreign `*.cfapps.us10-001.hana.ondemand.com` host and for a
    `*.applicationstudio.cloud.sap` host.
  - XSUAA matches a registered entry **exactly** — no implicit path wildcard. That is what makes
    the `ARC1_PUBLIC_URL` prefix guidance in the docs load-bearing, not optional.
- Added **R20** to the residual-risk register and extended the redirect-allowlist row of the
  per-PR review checklist, so the reason this list is narrow survives the next reviewer.
- Kept the runtime change, the descriptor narrowing and the UI helper's route handling. The
  helper's `routes` / plural `hosts` / `domains` support is upgrade compatibility for operators who
  already override the UI route, not speculative generality; the shipped default path produces the
  same descriptor before and after.
- Still not covered: a deployed ARC-1/AppRouter login and token exchange (CF route quota), and an
  installed IDE session. No completed security-plugin scan is claimed.

### Merge interactions

- `xs-security.json` conflicts with #813, which removes three lines from the list this PR replaces.
  Take this PR's list.
- #813's follow-up already removed the dependency-default assertion. Its descriptor guard
  remains compatible with this PR. The round trip over all seven supported manual callbacks
  lives in `oauth-redirect-policy.test.ts`; no temporary assertion still needs deleting.
- If this merges before 1.3.1 ships, add a row to the `## 1.3.1` section that #813 seeds, linking
  the upgrade table.

The existing runtime fix remains the simplest correction; no further runtime
change was justified. Full deployed ARC-1/AppRouter login and token exchange
remain unverified because of the route quota. No existing app or service was
modified. The temporary deployment is removed after the checks.

Follow-up review reran 30 production-route OAuth and MTA descriptor tests locally; all passed.
The R20 redirect-policy risk is supported by the route evidence. Full token-compromise impact
depends on the flow's consent, PKCE and token-exchange controls and was not established by the
callback tests. GitHub CI runs were excluded from this review.
