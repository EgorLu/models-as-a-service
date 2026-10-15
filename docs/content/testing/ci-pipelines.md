# Pipelines and execution

Where each suite actually runs, what triggers it, and which ones can block a merge.

## Gate matrix

| Pipeline | Platform | Trigger | Suites | Blocks merge |
|----------|----------|---------|--------|--------------|
| `maas-api-ci.yml` | GitHub Actions | PR touching `maas-api/**` | maas-api unit + lint | Yes |
| `maas-controller-ci.yml` | GitHub Actions | PR touching `maas-controller/**` and embedded manifests | maas-controller unit + lint | Yes |
| `build-test.yml` | GitHub Actions | PR touching controller api/cmd/pkg/Makefile/manifests | Build + manifest render | Yes |
| `e2e-collect.yml` | GitHub Actions | PR touching `test/e2e/**`, push to main | `pytest --collect-only` on Python 3.11 | Yes |
| `operator-chaos.yml` | GitHub Actions | PR touching controller api/controller/reconciler + deployment base | Operator chaos | Yes (path-filtered) |
| `openapi-validation.yml` | GitHub Actions | PR touching `maas-api/openapi3.yaml` | Spectral lint | Yes |
| `semgrep-tls.yml`, `tls-lint.yml`, `actionlint.yml`, `pr-title-validation.yml`, `disconnected-readiness.yml` | GitHub Actions | PR | Static checks | Yes |
| `approval-policy.yml` | GitHub Actions | PR + reviews | Advisory 1-vs-2 approval count by changed paths | Advisory (not yet a required check) |
| `update-payload-processing.yml` | GitHub Actions | Manual | Opens PR bumping the pinned aigpp commit + image | No (utility) |
| `scorecard.yml` | GitHub Actions | push main + weekly Mon 03:30 UTC | OpenSSF scorecard | No |
| `docs.yml`, `update-docs-latest.yml` | GitHub Actions | docs paths / manual | mkdocs build + publish | Docs only |
| `promote-main-to-stable.yml`, `promote-stable-to-rhoai.yml`, `create-release.yml` | GitHub Actions | Sunday 00:00 UTC + manual | Branch promotion, releases | No |
| **`maas-group-test`** | **Konflux (Tekton PAC)** | `pull_request` on main, `/group-test` comment, `group-test` event | **Full MaaS E2E on ephemeral Hypershift** | **Yes** |
| `early-gate-ci-build` / `early-gate-ci-test` | Konflux | `issue_comment` | Early-gate build + test | On demand |
| `odh-maas-{api,controller,discovery}-{pull-request,push,push-stable}` | Konflux | PR / push / push to stable | Image builds | Yes (build) |
| `ai-gateway-controller-group-test` | Konflux | PR on main, `/group-test` | AIGC + runtime-checkout MaaS E2E (Praxis gate) | Yes |
| `ai-gateway-payload-processing-e2e-ci-*` | Konflux | PR / push | AIGPP kind E2E | Yes |
| MaaS nightlies | Jenkins / ODH nightly | nightly | `opendatahub-tests` MaaS suite | No |
| ATRIS / Shepard | Jenkins (ATRIS org) | pipeline schedule | midstream `test/e2e` | No |
| Dashboard Cypress | GitHub Actions (odh-dashboard) | PR | `modelsAsAService` specs | Dashboard repo only |

---

## Konflux group test (the real E2E gate)

PAC `PipelineRun`: `.tekton/maas-group-test.yaml` in `models-as-a-service`.
Pipeline definition: `integration-tests/models-as-a-service/pr-group-testing-pipeline.yaml`
in [odh-konflux-central](https://github.com/opendatahub-io/odh-konflux-central), Tekton
`Pipeline` name `odh-pr-test-maas`.

It is a **group** test, not a per-component one: it builds a composite snapshot across
`odh-maas-api-ci` and `odh-maas-controller-ci` from the same PR, so the e2e run sees both
images together.

Task flow:

1. `generate-snapshot` — composite snapshot JSON for every component in `group-components`.
2. `audit-snapshot` — assert each row has image + `git.commit` + `git.url`; capture PR metadata from PAC annotations.
3. `provision-eaas-space` / `provision-cluster` — EaaS space and an ephemeral Hypershift cluster (`m5.2xlarge`).
4. `e2e-maas-openshift` (timeout 1h30m) — clone MaaS at the PR's source branch, export
   `MAAS_API_IMAGE` / `MAAS_CONTROLLER_IMAGE`, run `./test/e2e/scripts/prow_run_smoke_test.sh`.
   Runs with `onError: continue` so artifacts still get collected; a
   `deploy-and-e2e-status` file plus `fail-if-needed` re-assert the failure afterwards.
5. `must-gather` — `oc adm must-gather` into `ARTIFACT_DIR`.
6. `git-push-artifacts` — stage artifacts into `odh-build-metadata` branch `ci-artifacts`.
7. `finally` → `push-ci-artifacts-and-update-pr` — push OCI artifacts, comment the result
   on the PR, clean up the transient directory.

Per-component `IntegrationTestScenario` entries for MaaS were removed; everything goes
through the group pipeline on the `group-testing` / `maas-group` component.

### Reading a failure

- The PR comment from `pull-request-comment` links the artifact browser.
- junit XML and HTML report land in `ARTIFACT_DIR` (`e2e-<user>.xml`, `e2e-<user>.html`,
  plus a `-serial.xml` for the serial pass).
- must-gather output is in the same artifact tree.

---

## MaaS E2E runner mechanics

Entrypoint: `test/e2e/scripts/prow_run_smoke_test.sh` — a thin orchestrator that sequences
`deploy-platform.sh` → `deploy-models.sh` → `setup-test-tokens.sh` → `run_e2e_tests.sh`.

Three-pass execution: a parallel pass across xdist groups (`E2E_PARALLEL_WORKERS`,
default 7), then a serial pass for destructive modules, then an optional nightly pass
(`-m nightly`, only when `MAAS_PYTEST_INCLUDE_NIGHTLY=true`). Passes 1-2 always
exclude nightly-marked tests.

Key environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `SKIP_DEPLOYMENT` | `false` | Reuse an already-deployed cluster |
| `SKIP_VALIDATION` | `false` | Skip post-deploy validation |
| `MAAS_API_IMAGE` / `MAAS_CONTROLLER_IMAGE` | built-in | Image under test |
| `AI_GATEWAY_OPERATOR_IMAGE` | — | Requires `DEPLOY_MODE=operator` |
| `DEPLOY_MODE` | `kustomize` | `kustomize` or `operator` |
| `POLICY_ENGINE` | `rhcl` | Rate-limiting engine |
| `EXTERNAL_OIDC` | `false` | Enables external-OIDC tests (now largely superseded by the `nightly` marker tier) |
| `INSECURE_HTTP` | `false` | Deploy without TLS |
| `E2E_PARALLEL_WORKERS` | `7` | xdist workers |
| `MAAS_PYTEST_INCLUDE_NIGHTLY` | `false` | Run the extra nightly-marker pass (pass 3) |
| `DEPLOYMENT_NAMESPACE` | `opendatahub` | maas-controller namespace |
| `MAAS_SUBSCRIPTION_NAMESPACE` | `models-as-a-service` | MaaS CR namespace |
| `AITENANT_NAMESPACE` | `ai-tenants` | AITenant CRs |
| `GATEWAY_NAMESPACE` | `openshift-ingress` | Gateway |
| `MODEL_NAMESPACE` | `llm` | Model namespace |

### Local runs

```bash
# Full deploy + E2E (what CI does)
kustomize build test/e2e/fixtures | kubectl apply -f -
SKIP_DEPLOYMENT=true EXTERNAL_OIDC=false ./test/e2e/scripts/prow_run_smoke_test.sh

# Quick pytest only, cluster already up
./test/e2e/run-tests-quick.sh

# All modules locally
./test/e2e/smoke.sh
```

Unit tests:

```bash
make -C maas-api test
make -C maas-controller test
make -C maas-discovery test
```

PostgreSQL integration:

```bash
TEST_DATABASE_URL=postgres://... ./test/integration/postgresql.sh
```

---

## AI Gateway Controller Praxis gate

```bash
# Bump the pinned MaaS commit
MAAS_UPDATE_LOCK=true bash test/e2e/scripts/fetch-maas-e2e.sh

# Run it
AI_GATEWAY_CONTROLLER_IMAGE=quay.io/opendatahub/odh-ai-gateway-controller:odh-pr \
  ./test/e2e/scripts/prow_run_ai_gateway_controller_test.sh
```

Pipeline: `.tekton/ai-gateway-controller-group-test.yaml` → `odh-konflux-central`
`integration-tests/ai-gateway-controller/pr-group-testing-pipeline.yaml`, 4h timeout.

Until ATRIS and the `opendatahub-tests` nightlies switch to Praxis, **this is the primary
automated Praxis coverage**. Gaps are tracked under
[RHOAIENG-97459](https://issues.redhat.com/browse/RHOAIENG-97459).

---

## Nightlies (opendatahub-tests)

```bash
uv run pytest --collect-only tests/ai_gateway/models_as_a_service/
uv run pytest -m smoke tests/ai_gateway/models_as_a_service/
uv run pytest tests/ai_gateway/models_as_a_service/
```

These are signal, not a gate. A nightly failure files a `[maas] Failed test: ...` Jira
automatically; triage those against the inventory before assuming a product regression.

---

## Shift-left / TestOps

MaaS is **not** onboarded into `COMPONENTS_TESTS_CONFIG` on `rhoai-test-flow`
([RHOAIENG-84410](https://issues.redhat.com/browse/RHOAIENG-84410)). Until that lands,
"shift-left MaaS coverage" means the Konflux group test, not Jenkins.

To install a specific RHOAI build for manual testing, use the `rhoai-test-flow` Jenkins job
(`INSTALL_CLUSTER`, `ODS_BUILD_URL`, `UPDATE_CHANNEL=odh-nightly`, `ADD_ICSP=true`,
`RHODS_DEPLOYMENT_TYPE=Cli`). Enabling `RUN_TESTS` / `COMPONENTS_TESTS_CONFIG` there runs
the generic shift-left tiers, which currently do not include MaaS.
