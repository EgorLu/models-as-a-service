# Test inventory

Every suite that exercises MaaS or the AI Gateway components, grouped by repository.
Each entry lists the path, what it proves, how many tests it holds today, and who owns it.

---

## 1. models-as-a-service

Repo: [opendatahub-io/models-as-a-service](https://github.com/opendatahub-io/models-as-a-service)

### 1.1 Go unit tests

Counts as of 2026-10-15 (upstream main `0af16844`). Treat as orders of magnitude.
Re-verify with: `git grep -c "^func Test" <ref> -- maas-api | awk -F: '{s+=$NF} END {print s}'`
(same for maas-controller, maas-discovery) and `grep -c "def test_" test/e2e/tests/*.py | awk -F: '{s+=$NF} END {print s}'`.

| Package | Tests | What it covers |
|---------|-------|----------------|
| `maas-api/internal/api_keys` | 100 | Key minting, hashing, show-once, expiration, revocation, storage |
| `maas-api/internal/subscription` | 64 | Subscription selection, matching, tier resolution |
| `maas-api/internal/tlsprofile` | 33 | TLS profile parsing and cipher selection |
| `maas-api/internal/metrics` | 29 | Prometheus metric registration and cardinality |
| `maas-api/internal/auth` | 24 | Token validation, group extraction, identity sources |
| `maas-api/cmd` | 25 | Server wiring, CORS, flag parsing |
| `maas-api/internal/logger` | 17 | Structured logging, redaction |
| `maas-api/internal/token` | 15 | Token minting / JWT shape |
| `maas-api/internal/tenant` | 15 | Tenant resolution and scoping |
| `maas-api/internal/middleware` | 13 | Request middleware chain |
| `maas-api/internal/handlers` | 12 | HTTP handler contracts |
| `maas-api/internal/config`, `models`, `authpolicy`, `tracing` | 22 | Config load, model refs, AuthPolicy build, OTel |
| **maas-api total** | **369** | |
| `maas-controller/pkg/controller/maas` | 452 | Reconcilers for every MaaS CRD |
| `maas-controller/pkg/platform/tenantreconcile` | 117 | AITenant platform reconcile pipeline |
| `maas-controller/cmd/manager` | 31 | Manager setup, flags, RBAC wiring |
| `maas-controller/pkg/webhook` | 18 | Validating/defaulting webhooks |
| `maas-controller/pkg/reconciler/externalmodel` | 15 | ExternalModel / ExternalProvider reconcile |
| `maas-controller/pkg/oteljson`, `modelnaming`, `docsamples`, `api/...` | 23 | Telemetry JSON, naming, doc samples, CRD types |
| **maas-controller total** | **658** | |
| `maas-discovery/internal/tlsprofile` | 31 | TLS profile (shared logic) |
| `maas-discovery/internal/cache` | 24 | Discovery cache, gateway lookup |
| `maas-discovery/internal/gateway` | 21 | Gateway lookup/routing |
| `maas-discovery/internal/handler`, `middleware`, `auth`, `cert`, `types` | 30 | Discovery HTTP surface |
| **maas-discovery total** | **106** | |

Run: `make -C maas-api test`, `make -C maas-controller test`, `make -C maas-discovery test`.
Each produces `coverage.html` in its component directory **on success**; if any package
fails, make aborts before the HTML step and only a partial `coverage.out` exists.

!!! warning "Known issue (as of 2026-10-15, upstream main `0af16844`)"
    `make -C maas-api test` fails to build on a clean checkout: PR
    [#1343](https://github.com/opendatahub-io/models-as-a-service/pull/1343) changed the
    `ModelAccessChecker.AuthorizedModels` interface (now returns `(map, error)`) and
    updated `selector_test.go`, but `handler_test.go`'s `panickingModelAccessChecker`
    still implements the old signature. The PR's own merge checks did not run the
    maas-api unit job. Until fixed, `go test ./internal/subscription/` fails; the other
    14 packages pass. If you hit this, it's not your change.

### 1.2 PostgreSQL integration

`test/integration/postgresql.sh` — JSONB storage, GIN index queries, NULL handling against
a real PostgreSQL. Triple-gated so it never runs by accident: build tag `integration`,
not `testing.Short()`, and `TEST_DATABASE_URL` set.

### 1.3 E2E (pytest) — the shared suite

`test/e2e/tests/` — 34 test modules (~309 test functions) plus `test_helper.py`, a
helper named like a test module that contains none. This is the suite
`ai-gateway-controller` checks out at runtime (pinned in `test/maas-e2e.lock`), so
changes here ripple into another repo's merge gate.

Modules are assigned to **xdist groups**; each group gets a dedicated worker. Since
[#1638](https://github.com/opendatahub-io/models-as-a-service/pull/1638)
([RHOAIENG-82335](https://issues.redhat.com/browse/RHOAIENG-82335)) there is also a
`nightly` marker tier: nightly-marked tests are **excluded from PR /group-test runs**
and run in an extra pass only when `MAAS_PYTEST_INCLUDE_NIGHTLY=true`.

| Group | Module | Tests | Covers |
|-------|--------|-------|--------|
| `readonly` | `test_smoke.py` | 5 | Health, catalog shape, basic inference |
| `readonly` | `test_tenant.py` | 7 | MaasTenantConfig read-only validation |
| `readonly` | `test_config_tenant.py` | 5 | Config CR read-only validation |
| `readonly` | `test_tenant_discovery.py` | 5 | `/v1/tenants` shape |
| `readonly` | `test_networkpolicy.py` | 11 | NetworkPolicy + connectivity |
| `readonly` | `test_gateway_filter_chain.py` | 3 | Envoy filter chain ordering |
| `readonly` | `test_gateway_filter_chain_praxis.py` | 2 | Filter chain ordering under Praxis dataplane |
| `readonly` | `test_authpolicy_generation_stability.py` | 2 | AuthPolicy output stable over time |
| `api_keys` | `test_subscription.py` | 47 | Subscription enforcement, rate limits, auth flows |
| `api_keys` | `test_api_keys.py` | 40 | Key CRUD, admin authz, validation |
| `api_keys` | `test_subscription_list_endpoints.py` | 8 | Subscription listing endpoints |
| `api_keys` | `test_x_api_key_auth.py` | 5 | `X-API-Key` header auth (IPP ExternalModel) |
| `api_keys` | `test_embedding_inference.py` | 5 | Embeddings endpoint |
| `models` | `test_models_endpoint.py` | 22 | `/v1/models` subscription-aware filtering |
| `models` | `test_gateway_scoped_authpolicy.py` | 5 | Gateway AuthPolicy structure and lifecycle |
| `models` | `test_multi_tenant_integration.py` | 4 | Two-tenant coexistence |
| `models` | `test_model_identity_conflict.py` | 1 | MaaSModelRef identity collision |
| `security` | `test_negative_security.py` | 13 | Header spoofing, expired keys, cross-model access |
| `security` | `test_namespace_scoping.py` | 6 | Namespace isolation |
| `mt_lifecycle` | `test_tenant_namespace_discovery.py` | 11 | Namespace-label discovery, webhook validation (partly nightly) |
| `mt_lifecycle` | `test_aitenant_lifecycle.py` | 6 | AITenant create / migrate / delete |
| `mt_lifecycle` | `test_multi_tenant_maas_api.py` | 5 | Per-tenant Deployment, Service, HTTPRoute |
| `mt_lifecycle` | `test_crd_watch_resilience.py` | 3 | Dynamic CRD watch (destructive: restarts controller) |
| `tenant_isolation` | `test_tenant_auth_isolation.py` | 7 | Cross-tenant key rejection, OIDC scoping |
| `tenant_isolation` | `test_tenant_discovery_isolation.py` | 4 | Per-tenant `/v1/tenants` isolation |
| `tenant_isolation` | `test_tenant_rate_limit_isolation.py` | 2 | Per-tenant rate limit independence |
| `tenant_isolation` | `test_tenant_subscription_isolation.py` | 2 | Per-tenant subscription scoping |
| `tenant_ipp` | `test_per_tenant_ipp_isolation.py` | 10 | Per-tenant IPP stacks, routing isolation |
| `tenant_ipp` | `test_per_tenant_praxis_isolation.py` | 9 | Per-tenant Praxis dataplane isolation (new, #1636/#1656 era) |
| `tenant_inference` | `test_tenant_model_inference.py` | 6 | Cross-gateway inference isolation |
| `tenant_auto_resolve` | `test_tenant_auto_resolve.py` | 3 | MaaSModelRef auto-resolve of `tenantRef` |
| `external` | `test_external_oidc.py` | 28 | External OIDC flows (**nightly**, skipped in PR runs) |
| `external` | `test_external_models.py` | 9 | ExternalModel / ExternalProvider, egress, body routing |
| *(ungrouped)* | `test_trlp_rate_grouping.py` | 8 | TokenRateLimitPolicy rate-group behaviour |

Helpers (not tests): `conftest.py`, `test_helper.py`, `multitenancy_helpers.py`,
`worker_tenant_fixtures.py`.

### 1.4 Chaos / upgrade-breaking-change detection

`chaos/knowledge/maas.yaml` + `.github/workflows/operator-chaos.yml`. Uses the
[opendatahub-io/operator-chaos](https://github.com/opendatahub-io/operator-chaos) tool
(pinned by commit) against a declarative knowledge model of the controller's topology
(managed resources, webhooks, finalizers, steady-state checks). On every PR touching
controller API/pkg/reconciler, deployment manifests, or the knowledge model itself, the
workflow validates the model, runs local preflight, then diffs the PR's knowledge model
and CRD schemas against the base branch and **fails on breaking changes**. Purpose:
catch upgrade-breaking regressions at PR time (L1-L2 maturity today; L3-L4 future).
See `chaos/knowledge/README.md`.

---

## 2. ai-gateway-controller

Repo: [opendatahub-io/ai-gateway-controller](https://github.com/opendatahub-io/ai-gateway-controller)

| Suite | Path | Covers |
|-------|------|--------|
| Go unit + envtest | `pkg/tenant` (10), `pkg/render` (3), `pkg/controller` (2), `pkg/envelope` (2), `pkg/guardrail` (2), `pkg/publisher`, `pkg/resolver`, `api/...`, `test/envtest` (6) — 32 files | Reconcile logic, rendering, tenant resolution, guardrail; envtest covers controller against a real API server |
| kind ExtProc E2E | `test/kind-env/e2e.sh`, `e2e-extproc.sh`, plus `test-namespace-boundary.sh`, `test-provider-credentials.sh`, `test-suite-dispatch.sh` | Full ExtProc dataplane on kind, no OpenShift needed |
| External model static | `test/external-model/test-static.sh` | Static provider routing with templated manifests |
| Manifest helper tests | `hack/scripts/test-get-manifests.sh`, `test/e2e/scripts/test-deploy-ai-gateway-controller.sh` | Deploy tooling correctness |
| **Praxis E2E (runtime checkout of MaaS suite)** | `test/e2e/scripts/run_e2e_tests.sh` + `fetch-maas-e2e.sh` | **Today this is the primary automated Praxis gate** |

The suite is fetched into the gitignored `test/maas-e2e/` at the commit pinned in
`test/maas-e2e.lock`. An allowlist in
`run_e2e_tests.sh` decides which MaaS modules run. Excluded modules and their re-entry
criteria are documented in
[`test/e2e/TODO.md`](https://github.com/opendatahub-io/ai-gateway-controller/blob/main/test/e2e/TODO.md):
`test_external_oidc.py`, `test_x_api_key_auth.py`, `test_networkpolicy.py`,
`test_model_identity_conflict.py`, `test_tenant_auto_resolve.py`,
`test_crd_watch_resilience.py`, `test_authpolicy_generation_stability.py`.

!!! warning "Adding coverage to the Praxis gate"
    Landing a test in MaaS `test/e2e/tests/` is **not** enough. You also have to add it
    to the allowlist in `run_e2e_tests.sh` and bump `test/maas-e2e.lock`
    (`MAAS_UPDATE_LOCK=true bash test/e2e/scripts/fetch-maas-e2e.sh`).

---

## 3. ai-gateway-payload-processing (Praxis)

Repo: [opendatahub-io/ai-gateway-payload-processing](https://github.com/opendatahub-io/ai-gateway-payload-processing)

| Suite | Path | Covers |
|-------|------|--------|
| Go unit | `pkg/plugins/api-translation/translator/{openai,anthropic,azure,bedrock,vertex}`, `pkg/plugins/{apikey-injection,model-provider-resolver,nemo,external-metering,maas-headers-guard,passthrough-profile-picker,stream-usage-enforcer}`, `pkg/controller/{externalmodel,externalprovider,legacymigration,common}`, `pkg/dynamicmetadata` — 38 files | ExternalModel/Provider reconcile, legacy IPP migration, per-provider request translation, API-key injection, Nemo profile |
| kind E2E (ginkgo) | `test/e2e/e2e_test.go`, `e2e_suite_test.go`, driven by `test/e2e/scripts/e2e-ci.sh` + `setup-kind.sh` | ExtProc end-to-end on kind |
| Provider docs as spec | `docs/providers/*.md`, `docs/testing-with-simulator.md` | How to exercise each provider locally |

Reports from past runs live in `test/e2e/reports/<version>/`.

---

## 4. ai-gateway-operator

Repo: [opendatahub-io/ai-gateway-operator](https://github.com/opendatahub-io/ai-gateway-operator)

| Suite | Path | Covers |
|-------|------|--------|
| Go unit | `internal/controller/aigateway`, `pkg/config`, `api/components/v1alpha1`, `cmd/operator` (13 files) | AIGateway component reconcile, DSC config |
| Integration (envtest) | `test/integration/integration_test.go` | Controller against a real API server |
| E2E | `test/e2e/{ai_gateway,batch_gateway,models_as_service,e2e}_test.go` | Operator-driven install of AI Gateway, Batch Gateway, MaaS |
| Support harness | `test/support/{cluster,namespace,project}.go` | Shared cluster/namespace helpers |

Cleanup helpers: `hack/scripts/cleanup-e2e.sh`, `hack/scripts/cleanup-integration.sh`.

---

## 5. opendatahub-tests (product nightlies)

Repo: [opendatahub-io/opendatahub-tests](https://github.com/opendatahub-io/opendatahub-tests)
Path: `tests/ai_gateway/models_as_a_service/`

!!! note "Path moved"
    This suite used to live under `tests/model_serving/maas_billing/`. It is now
    `tests/ai_gateway/models_as_a_service/`. Older docs, tickets, and Jenkins configs
    pointing at the old path are stale.

| Area | Modules | Covers |
|------|---------|--------|
| `maas_subscription/` | 15 | Subscription enforcement, access control, filtering, rate-limit exemption, cascade deletion, multi-sub / multi-AuthPolicy, llm-d load balancing and prefix cache |
| `maas_api_key/` | 11 | Key CRUD, authorization, expiration, bulk ops, gateway rejection, tenant field, deny-by-default, `X-API-Key` |
| `multitenancy/` | 12 (`aitenant`, `isolation`, `inference`, `maas_api`) | AITenant bootstrap/cleanup, auth isolation, per-tenant maas-api routing |
| `body_base_routing/pre_auth_model_header/` | 5 | BBR Envoy filter, pre/post-auth inference, BBR rate limits |
| `external_model/` | 5 | Discovery, egress, auth, negative, cleanup |
| `component_health/` | 4 | maas-api, maas-controller, ai-gateway-controller, tenant health |
| `oidc_tests/` | 4 | Token flow, model access, multi-user, header injection |
| `observability/` | 2 | Metrics presence, enable path |
| `praxis/` | 2 | AITenant Praxis annotation, legacy IPP path |
| `upgrade/` | 3 | Pre/post-upgrade survival, ExternalModel legacy migration |
| `maas_cleanup/` | 1 | Post-run cleanup of MaaS resources |
| root modules | 6 | `/v1/models` + chat completions, RBAC E2E, request/token rate limits, token revoke, tokens |

Markers: `smoke`, `tier1`, `tier2`, `tier3`, `sanity`, `component_health`,
`pre_upgrade`, `post_upgrade`, `skip_on_disconnected`.

---

## 6. Suites owned by other teams

| Suite | Owner | Where | Notes |
|-------|-------|-------|-------|
| ATRIS / Shepard | ATRIS org | internal ATRIS pipeline definitions (`Jenkinsfile.maas-tests`, `pipelines/maas/config.yaml`, `run-maas-tests.sh` — held in ATRIS's internal repos, not public; path names per team lore, verify with ATRIS) | Runs `test/e2e/tests` from the **midstream** `red-hat-data-services/models-as-a-service` fork. Not on Praxis yet. Dashboard needs VPN. |
| Dashboard UI E2E | Dashboard team | `odh-dashboard` `packages/cypress/cypress/tests/e2e/maas/` (5 `.cy.ts` specs: Subscriptions, Policies, API keys, External models, Overview) + `gen-ai/testGenAiMaaS.cy.ts` + page objects at `pages/modelsAsAService.ts` | Subscription / AuthPolicy / API key CRUD, external models, overview |
| TestOps shift-left | TestOps / DevTestOps | Jenkins `rhoai-test-flow` → `COMPONENTS_TESTS_CONFIG` | MaaS **not onboarded yet** — spike [RHOAIENG-84410](https://issues.redhat.com/browse/RHOAIENG-84410) (Ishita) |

---

## Ownership

| Repo | Approvers (OWNERS) |
|------|--------------------|
| models-as-a-service | chaitanya1731, jland-redhat, dmytro-zaharnytskyi, SB159, ishitasequeira, yu-teo, somya-bhatnagar, liangwen12year, ryancham715, jrhyness, EgorLu, ahadas, mariusdanciu, pierDipi, bartoszmajsak, israel-hdez |
| ai-gateway-controller | nerdalert, jland-redhat, ishitasequeira, chaitanya1731, ryancham715, shaneutt, aslakknutsen, pierDipi, bartoszmajsak |
| ai-gateway-payload-processing | nirrozenbaum, nerdalert, jland-redhat, noyitz, yossiovadia, szedan-rh, chaitanya1731 |
| ai-gateway-operator | chaitanya1731, jland-redhat, ishitasequeira, jrhyness, somya-bhatnagar, pierDipi, bartoszmajsak, llm-d-team |

OWNERS files are the source of truth; the table above is a convenience snapshot.
