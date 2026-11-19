# Stable per-app names with Tailscale Services

Research for [#45](https://github.com/xkeanu/homeport/issues/45), checked
2026-11-19 against primary sources and Homeport `bf6b42e`.
**Live test: blocked.** No disposable tailnet or hosts were supplied. The local
Tailscale CLI failed at `tailscale version`: “Failed to load preferences.”
No policy, identity or Service configuration was changed.

## Recommendation

Keep Service creation and lifecycle in **Portless**, with explicit per-app
opt-in after the disposable-tailnet test. Homeport should consume the resulting
`tailscaleUrl`. Do not tie allocation to pinning: directory order and a shared
Service allowance have different lifecycles. This is a proposal; #45 requires
owner approval before product code.

The strongest case for building it here is a convenient stable-name toggle.
The cost is a second owner for startup, shutdown, recovery and names. Portless
already has [open PR #309](https://github.com/vercel-labs/portless/pull/309) for
Service sharing, explicit names, approval status and cleanup metadata. Its
author's manual-testing claim is supplied evidence, not our verification.
**Would change my mind:** upstream rejects lifecycle ownership, and a tested
Homeport design avoids competing with Portless.

## Answers to the ticket

1. **Limits and selection.** The February 19 GA announcement includes all
   plans, Personal included, with 10 Services and a beta-usage exception for at
   least a year. It invites requests for more. Treat 10 as the published
   baseline, not this account's verified allowance. **Recommendation:** opt
   in selected long-lived apps, counting existing Services first.
   [GA announcement](https://tailscale.com/blog/services-ga)
2. **Admin setup.** Define each Service in the console/API; approve its hosts
   or configure `autoApprovers.services`; grant client access separately.
   Hosts require tags and Tailscale ≥1.86.0; user-authenticated hosts are
   ineligible. Pre-authorized tagged hosts can advertise without fresh admin
   approval. [Setup and approval](https://tailscale.com/docs/features/tailscale-services#get-started)
   API automation uses authenticated, scoped credentials; the exact Services
   scope was not verified. **Recommendation:** keep admin credentials out of
   Homeport. A non-admin can operate an authorized host, but cannot provision
   the entire system alone. [API authentication](https://tailscale.com/docs/reference/tailscale-api#authentication)
3. **Ownership.** Portless main `6784e585a786a23b557d1291dfc86647587c4ea6`
   (package 0.18.1) still registers device ports through Serve/Funnel; #309
   proposes Service mode. Homeport copies `tailscaleUrl` into its page and
   peer snapshots. **Inference:** URL production belongs upstream; directory
   display may need no new field.
   [Upstream source](https://github.com/vercel-labs/portless/blob/6784e585a786a23b557d1291dfc86647587c4ea6/packages/portless/src/tailscale.ts),
   [proposal](https://github.com/vercel-labs/portless/pull/309),
   [page model](../server.mjs), [peer model](../peers.mjs)
4. **Multiple devices.** Service addresses are independent of hosts.
   Active/passive failover is the default; regional routing and MagicRoute
   require Premium/Enterprise. **Recommendation:** distinct explicit names,
   such as `app-device`, for independent environments; share one name only for
   interchangeable replicas. One app on two devices may consume two slots.
   [Service model](https://tailscale.com/docs/features/tailscale-services),
   [route selection](https://tailscale.com/docs/features/route-selection)
5. **Cleanup.** Service mode runs in background. Drain before clearing host
   configuration; deleting the definition is separate. **Inference:** killing
   the app is insufficient cleanup. Keep definitions across normal restarts
   for stable names, and retire them explicitly. DNS after drain, clear and
   deletion remains untested.
   [Lifecycle](https://tailscale.com/docs/features/tailscale-services#common-scenarios)

Tagging changes a device's authorization identity. It must never happen
automatically on a personal laptop. If apps and directory share a host, both
inherit the change; a separate tagged app host does not require tagging the
directory. Tagged devices cannot access individually shared nodes in another
tailnet, affecting peer aggregation. [Tags](https://tailscale.com/docs/features/tags),
[sharing restrictions](https://tailscale.com/docs/features/sharing),
[existing research](tailnet-sharing.md)

## Compatibility and source qualifications

- The issue describes definitions as policy entries. Current documentation
  puts definitions in the console/API; policy controls approval and access.
- Services docs say viewers ≥1.94 need no `accept-routes`; the GA post says
  ≥1.94.1. Use ≥1.94.1 for the experiment. Older Linux viewers need route
  acceptance. [Viewer setup](https://tailscale.com/docs/features/tailscale-services#step-4-access-tailscale-services-from-other-devices),
  [GA announcement](https://tailscale.com/blog/services-ga)
- L7 endpoints support authentication-header injection. Exact
  `Tailscale-User-Login` delivery and Homeport owner/viewer enforcement through
  a Service are untested. [Endpoint types](https://tailscale.com/docs/features/tailscale-services#endpoint-types)
- macOS remote-destination/multiple-interface caveats do not establish failure
  for loopback targets. Test actual variants and mobile viewers before
  promising support. [Limitations](https://tailscale.com/docs/features/tailscale-services#limitations)
- Upstream [#280](https://github.com/vercel-labs/portless/issues/280) reports
  stale ordinary Serve registrations. Those reports justify recovery checks;
  they do not prove Service mode has the same defect.

## Disposable-tailnet experiment: NOT RUN

This is an unqualified manual recipe, not an installation guide or acceptance
receipt. Use an explicitly authorized disposable tailnet, tagged test hosts A
and B, an allowed client and a denied client. Never re-tag a working machine.
Record OS, Tailscale variant/version on every device, Portless version/commit
and Homeport SHA. Keep policy backups and redacted results outside disposable
state. Retain no keys or tokens.

1. Inventory Services and Serve configuration. Allocate a unique run-owned
   name, e.g. `hp45-<run>`, and unused loopback backend ports. Record ownership
   before creation. An admin defines `tcp:443`, the host tag and its owners,
   approves A and grants only the allowed client access. Validate policy
   allow/deny tests first; avoid an inherited allow-all rule. Exercise
   auto-approval separately with a narrowly scoped test-host rule.
2. Start a synthetic backend returning marker A. Register
   `tailscale serve --service=svc:<name> --https=443 127.0.0.1:<port>`.
   From both clients record approval, DNS, TLS and HTTP results using bounded
   requests (`curl --max-time 10`). Allowed must receive A; denied must not.
   A zero CLI exit alone is not a successful connection.
3. Stop the backend; record failure behavior and retained advertisements.
   Drain (`tailscale serve drain svc:<name>`), wait for test connections to
   finish, then clear (`tailscale serve clear svc:<name>`). Restart on another
   backend port and register again. Require the same URL and correct marker.
   Test host/client restart too.
4. Advertise marker B on B under the same Service, approve it and record the
   routing mode. Drain A, then verify new requests reach B. Also test abrupt
   A outage. Record convergence time; repeated requests alone do not prove
   random load balancing.
5. With isolated Homeport fixtures, verify device and Service endpoints
   coexist on 443. Configure owners, exercise owner edits and viewer/no-login
   403s, and check stable links in pages and peer snapshots. Retest individually
   shared peers if the directory host is tagged. Expose no real project data.
6. Exercise the selected upstream candidate's normal exit, crash and recovery.
   Require owned advertisements to disappear or become visibly recoverable
   without altering another app. Direct CLI tests alone do not prove Portless
   integration.
7. Drain and clear only this Service on both hosts, stop owned backends,
   delete the run-owned definition using the supported console/API operation,
   and remove owned policy entries/devices. Record DNS after clear/deletion,
   including TTL/cache, and compare final inventory with baseline. Never use
   global `serve reset` for cleanup.

| Obligation | Result | Evidence required |
| --- | --- | --- |
| Setup, approval and allow/deny | BLOCKED | Versions, redacted policy tests, responses |
| Stable URL after restarts | BLOCKED | Same URL, new port, correct marker |
| Two-host routing and outage | BLOCKED | Mode, markers, convergence time |
| Owner/viewer and peer compatibility | BLOCKED | Allow/403 results, stable links |
| Upstream lifecycle and cleanup | BLOCKED | Candidate SHA, crash results, inventory |

Stop on unexpected exposure, unrelated changes or uncertain ownership.
Revoke test access and remove only established run-owned resources. Preserve
baseline resources and report cleanup failures. Failed obligations block
implementation acceptance; missing evidence remains unknown.

**#45 remains open** until the disposable-tailnet test is executed and the
owner records the design decision. This document authorizes no product code
or tailnet changes.
