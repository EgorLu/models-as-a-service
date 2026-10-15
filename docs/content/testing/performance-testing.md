# Performance testing

The honest state: **no automated performance or scale suite lives in any of the four
repos we own.** But MaaS *is* load-tested automatically — by PSAP, in their own
pipelines, since August 2026. Read the next two sections before anything else: the
first records what exists outside our repos, the second what exists inside them.

## What exists today

### Outside our repos: the PSAP CPT pipeline

PSAP (Vishnu Challa) has been running an automated MaaS performance pipeline since
August 2026. It is not in any MaaS repo; it is assembled from their tooling:

| Piece | Where | Notes |
|-------|-------|-------|
| Scheduled trigger | [openshift/release PR #83484](https://github.com/openshift/release/pull/83484) | Prow periodic (`maas-perf`) job, ROSA profile (3 masters + 3 workers) |
| Workload definition | [e2e-benchmarking `maas-gateway-perf`](https://github.com/cloud-bulldozer/e2e-benchmarking/tree/master/workloads/maas-gateway-perf) | Day-2 ops + benchmark driver; runs standalone via `run.sh` or from Prow |
| Load generator | kube-burner-ocp + GuideLLM K8s Jobs | A/B pairs: direct-to-simulator baseline vs gateway-routed, across providers x payload sizes x concurrency (16-1024) |
| Deploy under test | MaaS `deploy.sh` at a pinned `MAAS_REF` | Full stack: cert-manager, LWS, Kuadrant/RHCL, operator, DSC, PostgreSQL, gateway, controller, maas-api |
| Results | Elasticsearch/OpenSearch under run UUID + Grafana | [Dashboard](https://grafana.rdu2.scalelab.redhat.com:3000/d/dfvrhwyo2edxca/kube-burner-ocp-wrapper-vchalla-maas-experiments) (viewer/viewer) |
| Regression detection | Orion (Hunter change-point detection, 10% default threshold) | Auto-files Jira with run UUID, metrics, suspected PR; auto-closes on recovery |
| Querying | ChaiBot NL queries via Orion MCP backend | Answers "why did p99 go up last week" style questions |

Key facts for contributors: the pipeline works upstream (Prow) and downstream
(OpenShift CI is a downstream wrapper). Running the workload standalone needs an
existing OpenShift 4.19+ cluster with cluster-admin and `oc`; see the workload README
for the env-var matrix (`OPERATOR_TYPE`, `MAAS_REF`, `PROVIDERS`, `PAYLOAD_SIZES`,
`CONCURRENCY_LEVELS`). PR-triggered runs (every MaaS PR vs the nightly baseline) are
the direction PSAP wants next, and PR-run metadata is kept separate from baseline data
so the baseline stays clean.

Caveat for CI-number interpretation: PSAP uses ephemeral clusters; absolute numbers
are drift detection against yesterday's run, not capacity planning data. Same
philosophy as the Konflux scale-lane idea below.

### Inside our repos

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

No load generator (k6, locust, vegeta, GuideLLM) is wired into any pipeline *we* run;
no latency or throughput threshold gates any merge; no scale fixture exists for 1K+
subscriptions. (PSAP's pipeline above covers gateway-level A/B load, not the
subscription-scale fixtures this repo needs.)

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

The PSAP CPT pipeline (above) already provides, externally: a load generator, baseline
capture, scheduled runs, change-point regression detection, and automated Jira filing.
That raises the bar for what "building our own" means — anything we add should compose
with it rather than duplicate it. Three items remain ours to build:

1. **A scale fixture.** Parameterised N subscriptions / N models / N tenants, reusable
   from both the MaaS `test/e2e` harness and a standalone load job. Without this,
   everything else is bespoke. This is also the piece PSAP's gateway A/B runs do not
   cover — they exercise one deployed MaaS stack, not 1K+ subscription reconcile.
2. **A load generator decision.** GuideLLM is the RHOAI-aligned choice for inference load
   and is already blocked on [RHOAIENG-94562](https://issues.redhat.com/browse/RHOAIENG-94562).
   k6 or vegeta is lighter for pure gateway/auth-path load. Pick one for the auth path and
   one for the inference path; do not build two half-solutions. Whatever we pick should
   be runnable from kube-burner so the PSAP pipeline can drive it.
3. **A home for results we own.** PSAP's ES/Grafana/Orion stack is theirs; if the MaaS
   team needs gate-relevant numbers, they either live in PSAP's stack or in the Konflux
   artifact store. Decide which, deliberately, before building a third place.

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
