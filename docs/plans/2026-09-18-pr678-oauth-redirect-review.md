# PR #678: OAuth redirect review and implementation plan

## Decision and evidence

Continue [PR #678](https://github.com/arc-mcp/arc-1/pull/678), preserving its commits and
merging current main. The problem and the two-layer fix remain valid. A replacement PR would
lose useful attribution without changing the objective. Reviewed original head:
`6da2c2a3c6e4f95ea511163f18ad34780efa68bb`; main: `7a0016e9`.

The installed `@arc-mcp/xsuaa-auth` 1.0.2 and upstream main still include
`https://*.hana.ondemand.com/**` and `https://*.applicationstudio.cloud.sap/**` in the
manual client's fallback policy. Other tenants can own hosts under these domains. Its canonical
URL parser prevents the earlier parser-confusion bug (#387), but cannot make these other tenants
trusted. This is a distinct policy defect.

`src/server/http.ts` auto-registers a manual client's callback before the MCP SDK checks it.
The provider sends XSUAA ARC-1's `/oauth/callback`, with the final client URI inside signed state.
The callback handler checks the final URI again. Consequently, narrowing `xs-security.json`
alone cannot protect the second hop. Signing attacker-selected state is not an allowlist.
Code delivery to an untrusted callback is demonstrable locally; a production token compromise
also depends on login, client authentication and PKCE. Do not equate code delivery with a
verified production account takeover.

## Primary-source research

- [OAuth Security BCP, RFC 9700 §§2.1 and 4.1](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1):
  exact redirect matching is the default, with a native loopback port exception. Wildcards can
  admit attacker-controlled hosts even when the matcher is implemented correctly.
- [SAP XSUAA security considerations](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/f117cab6b92d438cb2a0b5204713994b.html):
  restrict `redirect-uris` as much as possible. SAP supports explicit wildcard syntax; support
  does not establish trust in unrelated tenants sharing a platform domain.
- [SAP security descriptor syntax](https://github.com/SAP-docs/btp-cloud-platform/blob/main/docs/30-development/application-security-descriptor-configuration-syntax-517895a.md):
  the service descriptor governs XSUAA's redirect check, not an application's subsequent redirect.
- [SAP CAP authentication guidance](https://cap.cloud.sap/docs/guides/security/authentication):
  deployment-provided application URLs can register the AppRouter's `/login/callback`.
- [MCP proxy security guidance](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/docs/2026-07-28/tutorials/security/security_best_practices.mdx):
  per-client consent is a separate defense when a proxy uses one upstream client. DCR's exact
  binding proves callback ownership within a registration, not user consent to that client.
  This PR must not claim to solve that broader proxy-consent problem.
- [Provider implementation](https://github.com/arc-mcp/xsuaa-auth/blob/main/src/redirect-uris.ts),
  also inspected in the installed package: ARC-1 can supply `redirectUriPatterns` directly.
  No dependency upgrade, custom matcher or new configuration framework is needed.

## Findings in the existing PR

1. Its explicit manual-client policy closes the shared-domain gap at the correct runtime boundary.
2. The AppRouter `host: arc1-ui-${space-guid}` changes existing URLs unnecessarily; the generator
   also overwrites an operator's chosen host. Export and consume the existing route instead.
3. Upstream `/**` permissions can be reduced to the actual callback and access-refresh paths.
4. The source-text assertion for production wiring is weaker than exercising `startHttpServer`.
   Test `/authorize`, `/register` and `/oauth/callback` through the actual Express middleware.
5. Documentation must preserve current deployment ownership guidance, explain gateway paths,
   and distinguish ARC-1 redirect rejection, XSUAA rejection and Entra's AADSTS50011.
6. The UI generator must preserve an operator's restrictive callback/grant settings. Adding the
   UI must not silently restore defaults the operator deliberately removed.

## Minimal plan

1. Merge main without rewriting published history. Retain current docs and unrelated fixes.
2. Keep the small explicit manual-client policy and existing provider matcher. Preserve supported
   fixed vendor/native callbacks and DCR exact binding; add no operator-facing setting.
3. Register only ARC-1 `/oauth/callback` and `/oauth/logged-out` under the deployment's URL.
   Keep local development callbacks in the standalone template. Export the optional AppRouter
   URL from its module and consume it only when the UI extension is enabled.
4. Reuse the shipped descriptor values in the UI generator; preserve explicit operator settings,
   append only the required AppRouter callback, and leave hosts/routes untouched.
5. Keep the recommended MTA path automatic. In the canonical XSUAA reference provide exact
   manual/gateway examples and an upgrade action; link there from existing task entry points.
6. Add HTTP regressions for foreign hosts, canonical-parser tricks, valid manual callbacks,
   DCR exact mismatch, state round trips, callback rejection and public URL prefixes. Add
   descriptor/generator coverage for base/UI, custom routes and restrictive overrides.
7. Run focused tests, all unit tests, typecheck, lint, build, policy/size gates, MTA validation
   and build, and strict docs build. Inspect generated deployment descriptors. Record live
   test evidence separately from local simulations; report any live deployment/login gap.
8. Review final diff for regressions, setup burden, documentation drift and scope. Fix and
   rerun affected checks. Push normal commits to #678 and update its description with evidence.

## Plan review before implementation

- No new feature flag, database, client registry, consent subsystem, or manual per-LLM setup.
- Default MTA and optional UI use existing routes. A custom gateway needs its actual two
  callback URLs because the deployer cannot infer an external reverse proxy.
- DCR signing labels, secrets, token policy, roles and tool schemas remain unchanged.
- Retained vendor/native wildcard paths are a bounded compatibility concession, not a claim of
  full RFC 9700 exact matching. Removing these established callbacks needs separate client evidence.
- Consent remains an independently scoped security question; document it explicitly rather than
  describing exact DCR callbacks as a complete confused-deputy defense.
- Tests must fail on main for the reported runtime issue and pass with the explicit policy.

## Execution and final review

The proposed module-reference simplification failed `npm run btp:build-ui-ext`: MBT refuses a
new `arc1-ui-api` requirement added to the XSUAA resource by an extension. A base requirement
would instead dangle when the optional module is omitted. Descriptor validation alone did not
catch this. Confirmed against MBT source (`removeUndeployedModules`) and the MultiApps dependency
validator. Revised plan: retain the PR's deployment-owned default UI hostname, preserve explicit
operator host/domain/routes, derive only those mapped callbacks, and document the default UI URL
migration. This avoids a custom build pipeline or another XSUAA service. The MCP route is unchanged.

### Validation results

- Before/after regression: temporarily removing the production `redirectUriPatterns` option
  makes three HTTP tests fail (foreign callback authorization, signed-state code forwarding,
  signed-state error forwarding). Restoring it passes all three.
- Final `npm test`: **225 files / 6,915 tests passed**. Includes 30 OAuth policy/HTTP cases,
  base descriptor contracts, generated UI defaults, idempotence, restrictive grant/callback
  overrides, custom host/domain, explicit multiple routes, and deployment documentation tests.
- `npm run typecheck`, `npm run lint`, `npm run build` (also in both MTAR builds),
  `npm run validate:policy`, `npm run check:sizes`, and `npm run docs:build`: passed.
  Lint reports three pre-existing informational suggestions in unrelated files, no errors.
- `npm run btp:validate`: all five combinations passed. This validates schema, not every
  extension merge. The supported flows additionally passed real `btp:build` and
  `btp:build-ui-ext`, and `mbt mtad-gen` for base, UI-only and the generated UI landscape
  extension. Inspected the emitted module lists, callbacks and dependency references.
- Two sibling extension files cannot be passed together to `mtad-gen` because they both extend
  `arc1-mcp`. The existing UI helper intentionally produces one combined extension; that
  supported path was tested. No new build/deploy command is required.

### Live XSUAA evidence

Created a temporary **unbound** XSUAA application service with the three exact callback paths,
using reserved `.invalid` test hosts. No existing app, service or role assignment was changed.
A temporary service key was kept in process memory, never printed or committed.

Unauthenticated authorization initially redirects all candidates to login, so that observation
alone cannot establish validation. Repeating `/oauth/authorize` with `prompt=none` distinguishes
the outcomes without a real user's session:

| Candidate | Observed result |
|---|---|
| Exact ARC-1 `/oauth/callback` | 302 to that URL with `error=login_required` |
| Exact ARC-1 `/oauth/logged-out` | 302 to that URL with `error=login_required` |
| Exact AppRouter `/login/callback` | 302 to that URL with `error=login_required` |
| Different path on the allowed host | 400, no Location |
| Unrelated CF host | 400, no Location |
| Unrelated BAS host | 400, no Location |
| Allowed hostname followed by an attacker suffix | 400, no Location |

Deleted the temporary key and service successfully after the probes. These observations verify
live redirect policy acceptance/rejection; they do not claim a browser login or token exchange.

### Final review and limits

Reviewed every changed file against main and traced the runtime change through the installed
provider, SDK authorization middleware, signed-state callback and DCR store. The smallest runtime
fix remains one explicit provider option plus a 20-line policy module. The rest is deployment,
regression coverage and setup guidance. No tool schema or LLM invocation changes.

The default MTA path needs only a full redeploy. Custom gateway operators register two exact
URLs; DCR clients register themselves. The optional UI's default hostname changes because of
MTA's dependency restriction; explicit routes survive the generated deployment extension.
Grant restrictions, token lifetimes, xsappname and unrelated resource config are preserved.
The canonical XSUAA guide is reachable from the existing BTP start/admin paths and includes
a decision table usable as raw Markdown or rendered HTML. This is a walkthrough review,
not a measured user-study claim.

No unresolved defect found within this patch's scope. Remaining test gap: no full MTA deployment
or authenticated browser login/token exchange through a deployed ARC-1 and AppRouter pair.
That requires a dedicated app deployment and an interactive identity session. Local HTTP tests
and live XSUAA policy probes cover the boundaries separately. Per-client DCR consent and the
retained vendor/native path patterns remain explicitly outside this targeted fix.
