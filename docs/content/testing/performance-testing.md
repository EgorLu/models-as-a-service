# Performance testing

The honest state: **there is no automated performance or scale suite in any of the four
repos today.** Everything we know about MaaS under load came from manual runs, PSAP
findings, and incident triage. This page records what exists, what we learned, and what
has to be built.

## What exists today

| Thing | Where | Status |
|-------|-------|--------|
| Controller performance tuning guidance | `docs/content/advanced-administration/controller-performance-tuning.md` | Documentation only |
| Subscription cardinality limits | `docs/content/advanced-administration/subscription-cardinality.md` | Documentation only |
| Telemetry cardinality guidance | `docs/content/advanced-administration/telemetry-defaults-and-cardinality.md` | Documentation only |
| Gateway resource workaround (PSAP-2514) | [RHOAIENG-71755](https://issues.redhat.com/browse/RHOAIENG-71755) | Doc PR in review |
| Replica-count knobs | maas-controller / tenant annotation | Shipped, untested at scale in CI |
| IPP autoscaling on CPU/memory | [RHOAIENG-79786](https://issues.redhat.com/browse/RHOAIENG-79786) | Shipped |
| Auth timeout 200ms → 2s | [RHOAIENG-79789](https://issues.redhat.com/browse/RHOAIENG-79789) | Shipped |
| GuideLLM against a MaaS endpoint | [RHOAIENG-94562](https://issues.redhat.com/browse/RHOAIENG-94562) | Blocked: `/health` not reachable through MaaS |

No load generator (k6, locust, vegeta, GuideLLM) is wired into any pipeline. No latency
or throughput threshold gates any merge. No scale fixture exists for 1K+ subscriptions.

## What we learned the hard way

Epic: [RHOAIENG-76579](https://issues.redhat.com/browse/RHOAIENG-76579) — General
performance and scale improvements.

| Finding | Ticket | State |
|---------|--------|-------|
| WasmPlugin CR hits the etcd size limit at 1K+ subscriptions | RHOAIENG-76583 | Closed |
| WASM auth does an O(N) route scan — degrades at 1K+ subscriptions | RHOAIENG-76585 | In progress |
| Authorino TTL cache stampede overloads maas-api at scale | RHOAIENG-76584 | Backlog |
| maas-controller took ~35s per subscription reconcile | RHOAIENG-76582 | Closed |
| Bidirectional gRPC stream per request blocks IPP streaming scale | RHOAIENG-79787 | New |
| Gateway pod OOMKilled under MaaS test load (Wasm plugin leak) | RHOAIENG-68937, RHOAIENG-68589 | Closed |
| MaaS returns HTTP 500 under load | RHOAIENG-71638 | Closed |
| Kuadrant pod OOMKilled | RHOAIENG-72393 | Closed |

Every one of these was found by a human running load by hand or by a customer/PSAP report.
None would have been caught by CI as it stands.

## The gap

We ship subscription-cardinality and replica-count knobs, and we document resource
workarounds, but nothing re-validates them. A regression that doubles reconcile time or
re-introduces the O(N) route scan passes CI green today.

What a shift-left performance capability needs, in order:

1. **A scale fixture.** Parameterised N subscriptions / N models / N tenants, reusable
   from both the MaaS `test/e2e` harness and a standalone load job. Without this,
   everything else is bespoke.
2. **A load generator decision.** GuideLLM is the RHOAI-aligned choice for inference load
   and is already blocked on [RHOAIENG-94562](https://issues.redhat.com/browse/RHOAIENG-94562).
   k6 or vegeta is lighter for pure gateway/auth-path load. Pick one for the auth path and
   one for the inference path; do not build two half-solutions.
3. **Baseline capture.** Record p50/p95/p99 for: token mint, `/v1/models`, auth decision,
   first-token latency. Store results as a CI artifact, the same way junit XML is stored
   by the group-test pipeline.
4. **Regression thresholds, non-blocking first.** Run on a schedule against main, report
   drift, do not gate. Only promote to a gate once the noise floor is known.
5. **A home.** The Konflux group test already provisions an ephemeral Hypershift cluster
   and publishes artifacts — that's the natural host for a scheduled scale lane, separate
   from the per-PR lane so it doesn't blow the 1h30m e2e budget.

## Why Konflux and not Jenkins

- The group pipeline already does cluster provisioning, image resolution from a composite
  snapshot, artifact publishing, and PR commenting. A scale lane reuses all of it.
- Jenkins shift-left (`COMPONENTS_TESTS_CONFIG`) is not onboarded for MaaS at all
  ([RHOAIENG-84410](https://issues.redhat.com/browse/RHOAIENG-84410)), so building
  performance there means solving onboarding first.
- Results stay next to the functional E2E artifacts, which is where anyone debugging a
  regression will already be looking.

The trade-off: ephemeral clusters are small (`m5.2xlarge`) and noisy neighbours are real.
Absolute numbers from CI will not match a sized production cluster. Treat CI numbers as
**drift detection against yesterday's CI numbers**, not as capacity planning data.

## Open questions

- Which cluster profile do we baseline on, and who pays for it to stay up?
- Do we gate on p95 regression, or only alert? (Start with alert.)
- Does the scale fixture live in MaaS `test/e2e` so aigc inherits it, or standalone?
- Who owns the baseline when it drifts for a legitimate reason?

Until these are answered, performance coverage is documentation plus incident response.
That should be stated plainly to anyone asking whether MaaS is performance-tested.
