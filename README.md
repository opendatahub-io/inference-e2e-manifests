# LLM-D Conformance Manifests — RHAIIS 3.5

LLMInferenceService manifests for RHAIIS 3.5 conformance testing.

Based on downstream samples: [red-hat-data-services/kserve (rhoai-3.5)](https://github.com/red-hat-data-services/kserve/tree/rhoai-3.5/docs/samples/llmisvc)

## What's New in 3.5 (from 3.4)

- **v1alpha2 API** — manifests use `serving.kserve.io/v1alpha2` (was `v1alpha1` in 3.4).
- **Scheduler nested under `router`** — the scheduler now lives at `spec.router.scheduler` alongside `router.route` and `router.gateway` (was top-level `spec.scheduler`).
- **No `tokenizer` sidecar in the scheduler** — the 3.5 scheduler preset (`config-llm-scheduler.yaml`) defines only the `main` (EPP) container. The `tokenizer` container stub used in 3.4 has been removed from all manifests.
- **New flow-control manifests** — `flow-control.yaml` and `flow-control-tokens.yaml` exercise the `flowControl` feature gate and saturation detection.
- **Cache-aware routing split into two plugins** — `precise-prefix-cache-producer` (KV event indexer) + `prefix-cache-scorer` (references the producer via `prefixMatchInfoProducerName`), replacing the single `precise-prefix-cache-scorer` used in 3.4.

## Branch Strategy

### Naming Convention

```
main                  ← always points to the latest GA release
├── <version>-GA      ← GA release branch, auto-synced from main via GitHub Actions
├── <version>-ea<N>   ← early access milestones (frozen after next EA or GA)
└── feature branches  ← short-lived, merged to main via PR
```

**Rules:**
1. `main` is the source of truth. All feature branches PR into `main`.
2. `<version>-GA` is auto-synced from `main` on every merge (see `.github/workflows/sync-main.yaml`). To change the active GA branch, update the `GA_BRANCH` env var in the workflow.
3. EA branches (`<version>-ea1`, `<version>-ea2`) are frozen once the next milestone is cut. Do not push to them after freeze.
4. To backport a fix to an older GA, cherry-pick from `main` into the target `<version>-GA` branch.
5. Delete feature branches after merge.

### Active Branches

| Branch | Version | Pull Secret | API | Status |
|--------|---------|-------------|-----|--------|
| `main` | RHAIIS 3.5 (latest) | `rhai-pull-secret` | `v1alpha2` | Active — PRs merge here |
| `3.5-GA` | RHAIIS 3.5 GA | `rhai-pull-secret` | `v1alpha2` | Auto-synced from main |
| `3.5-ea2` | RHAIIS 3.5 EA2 | `rhai-pull-secret` | `v1alpha2` | Frozen |
| `3.4-GA` | RHAIIS 3.4 GA | `rhai-pull-secret` | `v1alpha1` | Frozen |
| `3.4-ea2` | RHAIIS 3.4 EA2 | `rhaii-pull-secret` | `v1alpha1` | Frozen |
| `3.4-ea1` | RHAIIS 3.4 EA1 | `redhat-pull-secret` | `v1alpha1` | Frozen |

### CI Integration

The e2e test suite uses `--setup <branch>` to clone manifests from this repo:

```bash
uv run llm-d-e2e --setup main          # latest manifests
uv run llm-d-e2e --setup 3.5-GA        # GA-pinned manifests
uv run llm-d-e2e --setup 3.4-stable    # previous release
```

When cutting a new release (e.g., 3.6):
1. Create `3.6-ea1` from `main` for the first EA milestone
2. When GA ships, update `GA_BRANCH` in `.github/workflows/sync-main.yaml` to `3.6-GA`
3. The workflow will auto-create and sync the `3.6-GA` branch

## Key Notes

### Pull secrets

All manifests use `rhai-pull-secret`. This matches the `rhai-on-xks-chart` which creates `rhai-pull-secret` in all namespaces.

### Scheduler template with container stubs

When providing `router.scheduler.template.imagePullSecrets` in the manifest, you **must** also include the container name stub. Otherwise KServe replaces the entire template from `LLMInferenceServiceConfig` and the scheduler deployment fails with `spec.template.spec.containers: Required value`.

The scheduler preset (`config-llm-scheduler.yaml`) defines a single container, `main` (the EPP). In 3.5 there is no `tokenizer` sidecar in the scheduler template, so only the `main` stub is needed:

```yaml
router:
  scheduler:
    template:
      imagePullSecrets:
      - name: rhai-pull-secret
      containers:
      - name: main        # scheduler (EPP)
  route: {}
  gateway: {}
```

### Cache-aware plugin configuration (since 3.5)

Cache-aware routing now uses two cooperating plugins configured via `router.scheduler.config.inline`:

- `precise-prefix-cache-producer` — consumes vLLM KV cache events (ZMQ) and builds the block index.
- `prefix-cache-scorer` — scores endpoints, pointing at the producer with `prefixMatchInfoProducerName: precise-prefix-cache-producer`.

In 3.4 this was a single `precise-prefix-cache-scorer` plugin.

## Manifests

| Manifest | Description | Scheduler |
|----------|-------------|-----------|
| `single-gpu.yaml` | 1 GPU with scheduler | template + stubs |
| `single-gpu-smoke.yaml` | 1 GPU, minimal smoke test | template + stubs |
| `single-gpu-no-scheduler.yaml` | 1 GPU, K8s native routing (no EPP) | none |
| `cache-aware.yaml` | Prefix KV cache-aware routing, 2 replicas | template + stubs + config.inline |
| `flow-control.yaml` | Flow control feature gate, default request TTL | template + stubs + config.inline |
| `flow-control-tokens.yaml` | Flow control with token-based concurrency + saturation detection | template + stubs + config.inline |
| `tool-calling.yaml` | Tool-calling router stack | template + stubs + config.inline |
| `kv-offloading-cpu.yaml` | KV cache offloading, CPU RAM tier only | template + stubs + workload.kvCacheOffloading |
| `kv-offloading-tiered.yaml` | KV cache offloading cascade (GPU→CPU→disk), CPU + emptyDir | template + stubs + workload.kvCacheOffloading |
| `pd.yaml` | P/D disaggregation | template + stubs |
| `pd-cache-aware.yaml` | P/D + cache-aware hybrid | template + stubs + config.inline |
| `multi-pool.yaml` | Multi-pool VirtualService merge test | template + stubs |
| `moe.yaml` | MoE with DP/EP, 8 GPUs, RDMA/RoCE | template + stubs |
| `lora-single.yaml` | Single LoRA adapter (HF) | template + stubs |
| `lora-multi.yaml` | Multiple LoRA adapters with maxRank/maxAdapters | template + stubs |
| `maas-single-gpu.yaml` | `single-gpu` served through MaaS (see below) | template + stubs |
| `batch-gateway-async-sql.yaml` | vLLM-simulator `LLMInferenceService` + `LLMBatchGateway`, async batches over PostgreSQL (see below) | template + stubs |

### MaaS (RHOAI 3.5)

`maas-single-gpu.yaml` is the `single-gpu` LLMInferenceService bound to both
`inference-gateway` and `maas-default-gateway` (the MaaS controller only governs
routes on the MaaS gateway), plus:

| Resource | Namespace | Description |
|----------|-----------|-------------|
| `MaaSModelRef` | model namespace | Publishes the LLMInferenceService through MaaS |
| `MaaSAuthPolicy` | `models-as-a-service` | Grants authenticated users access to the model |
| `MaaSSubscription` | `models-as-a-service` | Token rate limit (100 tokens/1m) for authenticated users |

The policy and subscription reference the model in `llm-conformance-test`. The
cluster must have the RHOAI 3.5 MaaS components (RHCL/Kuadrant, `maas-api`) and
both gateways installed.

### Batch Gateway (`batch-gateway-async-sql.yaml`)

One `LLMInferenceService` plus one `LLMBatchGateway` that serves the OpenAI
Batch API (`/v1/files`, `/v1/batches`), using async dispatch over PostgreSQL SQL
queues — Postgres holds both the control-plane state and the request/result
queues, so no Redis is involved. The documents, in apply order:

| Resource | Description |
|----------|-------------|
| `LLMInferenceServiceConfig` | vLLM inference simulator (`llm-d-inference-sim` v0.12.0, the version RHOAI 3.6 ships): CPU-only, no model weights, TLS from the controller-mounted certificate |
| `LLMInferenceService` | Model under test: the simulator, served via the config's `baseRef`; no GPU |
| `batch-gateway-postgres` Service + StatefulSet | Control-plane state and SQL async queues (`emptyDir`, disposable). The volume is mounted at `/var/lib/postgresql`, not at `PGDATA`: under OpenShift's random UID, `initdb` cannot `chmod` a kubelet-owned volume root and the pod crash-loops. |
| `batch-gateway-seaweedfs` Deployment + Service | S3-compatible file storage for batch input/output files |
| `batch-gateway-secrets` Secret | `postgresql-url`, `s3-secret-access-key`, `inference-api-key` |
| `LLMBatchGateway` | API server, processor (`modelGateways` + `asyncConfig.sql`), GC |

The test suite deletes these in reverse order, so the `LLMBatchGateway` stays
last.

`spec.processor.asyncConfig.inferenceGateway.url` is the one field that does not
port across platforms, so the conformance suite rewrites it at deploy time
(along with any `sql.queuesConfig[].igwBaseURL`) from the gateway Service it
discovers via the `gateway.networking.k8s.io/gateway-name` label:

| Platform | Derived dispatch URL | TLS the suite sets |
|----------|----------------------|--------------------|
| RHOAI 3.6, AIGateway component | `https://data-science-gateway-<class>.<ingress-ns>.svc.cluster.local:443/<test-ns>/<llmisvc>` | `asyncConfig.tls.insecureSkipVerify: true` — the gateway certificate is signed by the OpenShift service CA, which the batch components do not trust |
| Install that still exposes Istio's gateway | `http://inference-gateway-istio.redhat-ods-applications.svc.cluster.local:80/<test-ns>/<llmisvc>` | none (the manifest's `tls` block is left as written) |

The value checked in here is therefore only a starting point for a manual
`kubectl apply`. The suite's own override is
`validation.batchGateway.dispatchUrl` in the test case, which is used verbatim
and skips discovery. Find the gateway Service with:

```bash
kubectl get svc -A -l gateway.networking.k8s.io/gateway-name \
  -o custom-columns=NAME:.metadata.name,NS:.metadata.namespace,PORTS:.spec.ports[*].port
```

Platform requirement: the `batch.llm-d.ai` CRD and the batch gateway operator
must be enabled on the cluster
(`spec.components.aigateway.batchGateway.managementState: Managed` on the
`DataScienceCluster` — see the conformance suite's
`docs/adding-a-batch-test-case.md`). Nothing beyond the backing Postgres and S3
stores is installed by this manifest.
