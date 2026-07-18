# Research: Cloud-Native Deployment Options for TuFT

*Status: research note / RFC draft — 2026-07-18*

This note surveys deployment options for TuFT beyond the two existing backends
(Modal and Lambda Cloud), targeting the roadmap item:

> **Cloud-native deployment**: Integration with AWS, Alibaba Cloud, GCP, Azure
> and Kubernetes orchestration.

It ends with a concrete recommendation for the next step.

**TL;DR.** Kubernetes is the only substrate that covers all four clouds with a
single integration — the multi-cloud launchers (SkyPilot, dstack) both skip
Alibaba Cloud, and nearly all managed AI platforms are structural mismatches
for a stateful server (the one exception: Alibaba PAI-EAS). Recommended next
step: **ship a Helm chart (`deploy/kubernetes/`) validated end-to-end on one
managed cluster**, then thin per-cloud guides (ACK/EKS/GKE/AKS), then a
SkyPilot helper for the no-cluster VM path. Details in §8–9.

## 1. Where we are today

TuFT ships two deploy helpers, both thin wrappers around the same standard
server (`tuft launch`) and the same published image
(`ghcr.io/agentscope-ai/tuft:latest`):

| Backend | Model | Mechanism |
|---|---|---|
| [Modal](../sphinx_doc/source/deployment/modal.md) | Serverless, scale-to-zero | `deploy/modal/launch.py` renders a Modal app from `tuft_config.yaml`; volumes for HF cache + checkpoints |
| [Lambda Cloud](../sphinx_doc/source/deployment/lambda.md) | On-demand GPU VM | `deploy/lambda/launch.py` calls the Lambda REST API; cloud-init runs `docker run` on first boot |

Both follow the same conventions, which any new backend should preserve:

1. **One portable config file.** Users edit a standard `tuft_config.yaml`; the
   backend's infra knobs live in a provider section (`modal:`, `lambda:`) that
   is stripped before the server sees the config. The same file works with
   `tuft launch --config` locally.
2. **Single-file helper with verbs.** `deploy/<backend>/launch.py` supports
   launch / `--down` / `--status` / `--dry-run`. No Python edits required.
3. **One documented end-to-end example.** Each guide walks through the same
   "talk like Yoda" SFT run on `Qwen/Qwen3-0.6B` and adapter download.
4. **Cheap E2E CI.** `.github/workflows/deploy-e2e.yml` deploys the published
   image with a small model on the cheapest GPU, health-gates
   `/api/v1/healthz`, checks `get_server_capabilities`, and always tears down.

## 2. The deployment contract (extracted from the code)

Whatever the target infrastructure, TuFT needs:

| Requirement | Detail | Source |
|---|---|---|
| Container | `ghcr.io/agentscope-ai/tuft:latest`, CUDA 13 base (`nvcr.io/nvidia/cuda:13.0.2`) | `docker/Dockerfile` |
| GPUs | 1+ NVIDIA GPU (A100/H100 class recommended; `colocate: true` fits one model on one GPU) | `deploy/modal/tuft_config.example.yaml` |
| Port | One HTTP port (10610), API-key auth (`tml-` keys) | `src/tuft/server.py`, deploy helpers |
| Durable storage | A `/data`-style volume for the HF model cache and `checkpoint_dir` | both deploy helpers |
| Shared memory | Large `/dev/shm` (64–128 GB; `docker --shm-size`) for torch/vLLM workers | `deploy/lambda/launch.py`, README |
| Secrets | Optional `HF_TOKEN` for gated models; API keys in the config | both deploy helpers |
| Health gate | `/api/v1/healthz` returns 200 only after models are loaded — use as readiness probe | `deploy-e2e.yml` |
| **Statefulness** | **Session/run state lives in one process — exactly one replica** (`max_containers: 1` on Modal); optional Redis/file persistence for crash recovery | `deploy/modal/launch.py`, `config/tuft_config.example.yaml` |
| Ray in-process | The server starts Ray actors internally: vLLM engines (fractional `num_gpus`) and verl/FSDP training workers (`num_gpus=1` each). Today: one node; multi-node is a separate roadmap item | `src/tuft/backends/*.py` |

Two consequences worth stating explicitly:

- TuFT is a **long-running stateful server**, not a batch training job. Products
  built around submit-a-job semantics (SageMaker Training, PAI-DLC, Vertex AI
  custom training, Kubeflow) are a structural mismatch; products built around
  serving a container behind an endpoint are a fit.
- Because the server manages its own Ray actors on the GPUs it sees, the
  infrastructure's job is simple: give one container N GPUs, a volume, a port,
  and big `/dev/shm`. That makes the integration surface small and portable.

## 3. Evaluation criteria

For each option we ask:

1. **Coverage** — how many of the roadmap targets (AWS, Alibaba, GCP, Azure,
   K8s) does one integration cover?
2. **Fit** — can it run a stateful single-replica GPU container with a volume,
   an exposed port, and configurable shm?
3. **Maintenance cost** — how much provider-specific code do we own?
4. **User cost model** — scale-to-zero / per-second billing (Modal-like) vs.
   run-until-terminated (Lambda-like)?
5. **China accessibility** — Alibaba Cloud and mirror-friendliness matter for a
   core part of the user base (the Dockerfile already carries Alibaba PyPI and
   HF mirror hooks).

## 4. Option A — Kubernetes first (the common denominator)

All four roadmap clouds sell managed Kubernetes — **EKS** (AWS), **GKE** (GCP),
**AKS** (Azure), **ACK** (Alibaba) — and it is the *only* substrate that covers
all four (plus on-prem GPU clusters) with a single artifact. TuFT's contract
maps onto small, boring K8s primitives:

| Contract item | K8s realization |
|---|---|
| One stateful replica | `Deployment` with `strategy: Recreate` (old pod releases the GPU + RWO PVC before the new one starts) — or a 1-replica StatefulSet; prior-art LLM charts use Deployments |
| GPUs | `resources.limits: {nvidia.com/gpu: N}` via the NVIDIA device plugin; the GPU Operator is only needed on self-managed nodes (managed clouds preinstall drivers — see below) |
| Big `/dev/shm` | pods default to 64Mi shm → mount `emptyDir: {medium: Memory, sizeLimit: 64Gi}` at `/dev/shm`; tmpfs counts against the container memory limit, so size limits ≥ shm + RSS |
| `/data` volume | RWO PVC, `volumeBindingMode: WaitForFirstConsumer`; default storage classes exist on all four clouds (EBS gp3 / PD-Hyperdisk / Azure Disk / Alibaba cloud disk) |
| Slow startup (model load) | `startupProbe` on `/api/v1/healthz` with a large budget (e.g. `periodSeconds: 15, failureThreshold: 60` ≈ 15 min), then readiness + conservative liveness |
| Secrets | `Secret` env refs for `HF_TOKEN` + API keys, `existingSecret` support in values; External Secrets Operator for cloud secret managers later |
| Exposure | ClusterIP + Ingress or `type: LoadBalancer`; `kubectl port-forward` for the SSH-tunnel-style dev flow the Lambda guide already teaches |
| Optional Redis persistence | standard Redis subchart / managed Redis endpoint in `persistence.redis_url` |

Sources: [k8s emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir),
[Deployment strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy),
[probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/),
[NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html).

### Managed K8s GPU state per cloud (verified 2026-07)

- **EKS**: Auto Mode bundles managed Karpenter — a `nvidia.com/gpu` request
  auto-provisions GPU nodes on Bottlerocket accelerated AMIs (driver, runtime,
  device plugin preinstalled). On non-Auto clusters, AL2023 accelerated AMIs
  ship the driver but **not** the device plugin (install it or the Operator).
  ([AWS](https://docs.aws.amazon.com/eks/latest/userguide/ml-node-pools.html))
- **GKE**: Standard node pools auto-install drivers + plugin; **Autopilot**
  runs GPU pods fully managed (B200/H200/H100/A100/L4/T4 via
  `cloud.google.com/gke-accelerator` selector). Dynamic Workload Scheduler
  *flex-start* gives discounted on-demand GPU capacity for runs up to 7 days.
  ([GKE Autopilot GPUs](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/autopilot-gpus),
  [DWS](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/dws))
- **AKS**: the recommended path is now AKS-**managed GPU node pools** (AKS owns
  driver/plugin/DCGM). Azure also ships **KAITO** (AI toolchain operator
  add-on, CNCF sandbox) that auto-provisions right-sized GPU nodes from a
  Workspace CRD — prior art worth imitating, not a dependency.
  ([MS Learn](https://learn.microsoft.com/en-us/azure/aks/use-nvidia-gpu),
  [KAITO](https://github.com/kaito-project/kaito))
- **Alibaba ACK**: GPU node pools via `ack-ai-installer`; **cGPU** kernel-level
  GPU sharing (memory isolation, ACK Pro) is unique among the four clouds and
  aligns with the "serverless GPU / multi-tenant sharing" roadmap item. ACS
  (successor of ACK Serverless) documents serverless GPU pods (48/96/141 GB
  cards), though GPU compute there still appears to be invitational preview
  (GA status unverified).
  ([cGPU](https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/cgpu-overview/),
  [ACS pods](https://www.alibabacloud.com/help/en/cs/user-guide/acs-pod-instance-overview))

### Prior art for the chart

- [vLLM production-stack](https://github.com/vllm-project/production-stack):
  values-driven per-model Deployment + Service + PVC — good template for GPU /
  HF-token / PVC knobs.
- [otwld/ollama-helm](https://github.com/otwld/ollama-helm): community chart
  closest to TuFT's shape (single Deployment + PVC + GPU toggle + model
  bootstrap).
- [KubeAI](https://github.com/substratusai/kubeai) (operator + Model CRD,
  scale-from-zero) and [KAITO](https://github.com/kaito-project/kaito)
  (CRD auto-provisions GPU nodes): patterns for a *later* TuFT operator;
  overkill for v1. [llm-d](https://llm-d.ai) targets disaggregated inference
  gateways — not our shape.

### Multi-node later (ties into the distributed-training roadmap item)

TuFT starts Ray in-process today, so plain K8s suffices now. When multi-machine
training lands, [KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
(mature; RayCluster/RayJob/RayService CRDs) is the natural fit — a `RayCluster`
with the TuFT server as head-pod entrypoint. (RayService targets Ray Serve
apps, not FastAPI+actors.) [LWS](https://github.com/kubernetes-sigs/lws) is an
alternative if only engines span nodes; Kueue matters only for shared-quota
queueing; JobSet is batch-shaped and irrelevant for an always-on server. A v1
chart should not depend on any of these — just not paint itself into a corner
(e.g., keep the server pod spec reusable as a head-pod template).

### GPU sharing (serverless-GPU roadmap item)

Node-level sharing — time-slicing (no isolation), MPS (fraction enforcement,
fragile), MIG (hardware isolation, A100/H100+), Alibaba cGPU — is configured
via device-plugin/Operator config, orthogonal to the chart. TuFT already packs
tenants *inside* one process, so node-level sharing mainly helps carve big
GPUs into multiple small TuFT instances, or co-locate TuFT with other
workloads (MIG safest). K8s DRA (GA in v1.34) is the forward-looking API here.
([NVIDIA GPU sharing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html))

## 5. Option B — Multi-cloud launchers (SkyPilot, dstack)

These frameworks let one YAML target many clouds, replacing N per-cloud
scripts with one integration. Verified state as of 2026-07-18:

| | SkyPilot v0.13 | dstack 0.20 |
|---|---|---|
| AWS / GCP / Azure | ✅ / ✅ / ✅ | ✅ / ✅ / ✅ |
| **Alibaba Cloud** | ❌ (not supported) | ❌ (not supported) |
| Lambda Cloud | ✅ | ✅ |
| Kubernetes | ✅ | ✅ |
| Other infra | OCI, Nebius, Crusoe, CoreWeave, RunPod, Vast.ai, DigitalOcean, Paperspace, Slurm, vSphere, … | OCI, Nebius, Crusoe, Vultr, RunPod, Vast.ai, Slurm, on-prem SSH fleets, … |
| License / community | Apache-2.0, ~10.3k stars | MPL-2.0, ~2.2k stars |
| Persistent volumes | K8s-native `volumes`; on VM clouds only bucket `file_mounts` or the stopped VM's disk | Network volumes on AWS/GCP/RunPod/K8s only (not Azure/Lambda); instance volumes elsewhere |
| Service exposure | `resources.ports` opens firewall + endpoint | `type: service` + `port`, optional gateway for HTTPS, token auth built in |
| Scale-to-zero | VM autostop; SkyServe `min_replicas: 0` exists but SkyServe is beta, not recommended for production | `replicas: 0..N` + RPS-based autoscaling (needs gateway) |
| `--shm-size` | via `docker.run_options: [--shm-size=128g]` | first-class `resources.shm_size` field |

Sources: [SkyPilot supported infra](https://docs.skypilot.co/en/latest/getting-started/installation.html),
[SkyPilot YAML spec](https://docs.skypilot.co/en/latest/reference/yaml-spec.html),
[dstack backends](https://dstack.ai/docs/concepts/backends/),
[dstack services](https://dstack.ai/docs/concepts/services/),
[dstack volumes](https://dstack.ai/docs/concepts/volumes/).

**SkyPilot caveats specific to TuFT:**

- **Ray conflict (real risk, manageable).** SkyPilot runs its own internal Ray
  (port 6380) on every node it provisions. Apps may run their own Ray, but must
  never call `ray.init(address="auto")` (would join SkyPilot's cluster) and
  never `ray stop`. TuFT's in-process `ray.init()` is compatible as long as we
  document/guard this.
  ([SkyPilot FAQ](https://docs.skypilot.co/en/latest/reference/faq.html),
  [Ray-on-SkyPilot example](https://docs.skypilot.co/en/stable/examples/training/ray.html))
- **Durability.** No cross-cloud block-volume abstraction: on VM clouds, /data
  persistence means bucket mounts (`MOUNT_CACHED` for checkpoints) or relying
  on autostop-*stop* (not down) keeping the disk.
- SkyPilot is now a client-server system (background local API server) — one
  more moving part in a deploy helper, but also enables team deployments.

**dstack notes:** the `type: service` concept maps almost 1:1 onto TuFT's
contract (image, port, replicas: 1, volume, HTTP probe on `/api/v1/healthz`,
built-in token auth in front of TuFT's own keys), and `shm_size` is first-class.
Smaller community; volume gaps on Azure/Lambda.

**Ray's own cluster launcher** has a community-maintained Aliyun backend, but it
provisions bare Ray clusters — architecturally colliding with TuFT's in-process
Ray — and doesn't manage services/volumes/ports as a product. Poor fit.

**Key structural fact:** a launcher integration covers AWS + GCP + Azure (+
Lambda, RunPod, Nebius, …) in one shot, but **Alibaba Cloud is reachable only
through the Kubernetes backend (i.e., ACK)** or a bespoke integration.

## 6. Option C — Native per-cloud paths (outside Kubernetes)

Verified state per cloud (2026-07-18). Two sub-options each: the plain
**GPU VM + cloud-init** path (parity with the existing Lambda helper) and the
cloud's **managed AI platforms**.

### The VM path works everywhere — but costs 4× maintenance

All four clouds expose an API-drivable "launch instance with user-data" flow,
so `deploy/lambda/launch.py` could be ported almost mechanically:

| Cloud | API | user-data | Single-GPU instance families |
|---|---|---|---|
| AWS | EC2 `RunInstances` (boto3) | cloud-init | g6e (L40S), p4d (A100), p5 (H100); [Capacity Blocks](https://aws.amazon.com/ec2/capacityblocks/) reserve scarce H100/H200 and run the same user-data |
| GCP | `instances.insert` | `startup-script` metadata | G2 (L4), A2 (A100), A3 (H100/H200), A4 (B200) |
| Azure | `Microsoft.Compute/virtualMachines` | cloud-init (`customData`) | NC_A100_v4, NCads_H100_v5 (H100 NVL 94 GB — best single-GPU fit) |
| Alibaba | ECS `RunInstances` (`UserData` param, [docs](https://www.alibabacloud.com/help/en/ecs/user-guide/manage-the-user-data-of-linux-instances)) | cloud-init | gn7e (A100 80 GB), gn8is (L20), gn8v (H20) |

Four more `launch.py` scripts would each need auth, instance-type discovery,
capacity fallback, teardown verbs, docs, and E2E CI — the maintenance profile
that multi-cloud launchers (§5) and Kubernetes (§4) exist to avoid.

### Managed AI platforms: almost all structural mismatches

TuFT is a long-running stateful server with its own multi-route API and its
own auth — which breaks nearly every managed inference/training product:

| Platform | Verdict | Reason |
|---|---|---|
| SageMaker real-time endpoints | ❌ | fixed contract: port 8080, only `/invocations` + `/ping`, ~60 s response timeout ([docs](https://docs.aws.amazon.com/sagemaker/latest/dg/your-algorithms-inference-code.html)) |
| SageMaker Training / PAI-DLC / Vertex custom training | ❌ | batch-job semantics, no inbound HTTP — TuFT is not a job |
| SageMaker HyperPod | ⚠️ | resilient GPU clusters, but orchestration is Slurm or **EKS** — i.e., it merges into the K8s path |
| Vertex AI prediction (custom container) | ❌ | single `predictRoute`/`healthRoute`, stateless autoscaled replicas ([docs](https://docs.cloud.google.com/vertex-ai/docs/predictions/use-custom-container)) |
| Cloud Run GPU | ❌ | GA with scale-to-zero, but L4 / RTX PRO 6000 only (no A100/H100) and ephemeral instances ([docs](https://docs.cloud.google.com/run/docs/configuring/services/gpu)) |
| Azure ML online endpoints | ❌ | scoring-route model, `request_timeout_ms` ≤ 180 s |
| Azure Container Instances GPU | ❌ | **retired July 2025** ([docs](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-gpu)); successor Azure Container Apps serverless GPU is scale-to-zero/stateless-shaped |
| ECS on EC2 GPU | ⚠️ | workable long-running service middle ground (Fargate still has no GPUs), but AWS-only — inferior to EKS for us |
| **Alibaba PAI-EAS** | ✅ | **the one genuine fit**: deploys an arbitrary custom-image GPU HTTP service (your image, command, port), fronted by EAS auth/gateway, with OSS/NAS mounted into the container for `/data` ([custom-image deploy](https://www.alibabacloud.com/help/en/pai/user-guide/deploy-a-model-service-by-using-a-custom-image)) |
| Alibaba Function Compute GPU | ⚠️ | closest Alibaba Modal-analog (scale-to-zero via instance freezing) but T4/A10/L20 only (≤48 GB) and freezing pauses background work ([docs](https://www.alibabacloud.com/help/en/functioncompute/fc/user-guide/real-time-inference-scenarios-1)) |

**Takeaway:** don't chase the managed inference platforms — they fail on fixed
route contracts, short timeouts, and statelessness. The two per-cloud things
worth doing eventually are a **PAI-EAS guide** (best managed fit anywhere in
this survey, and squarely aimed at the China audience) and, only if demand
shows up, VM scripts — which SkyPilot/dstack mostly obsolete outside Alibaba.

## 7. The rest of the GPU-cloud landscape (secondary)

Not roadmap targets, but users keep asking for "the Modal/Lambda of X".
Verified verdicts (2026-07-18) on whether each could run TuFT's stateful
long-session server:

**Modal-like serverless platforms:**

| Provider | Verdict | Why |
|---|---|---|
| [Beam](https://docs.beam.cloud/v2/pod/web-service) | ✅ closest Modal analog | Pods run arbitrary images with exposed ports, persistent Volumes, persistent HTTPS endpoints; open-source backend |
| [Cerebrium](https://docs.cerebrium.ai/cerebrium/storage/managing-files) | ⚠️ partial | custom Dockerfile + 50 GB persistent storage, but request-driven autoscaling; long-session tolerance unverified |
| [Koyeb GPU](https://www.koyeb.com/docs/reference/volumes) | ⚠️ partial | always-on GPU services with scale-to-zero, but volumes are preview-only, 10 GB max |
| RunPod Serverless | ❌ | queue-based request/response workers, 24 h cap — fights a stateful server |
| Baseten | ❌ | inference-shaped custom servers, no persistent volume |
| Replicate | ❌ | Cog predict/train API only, no arbitrary web server |

**Lambda-like on-demand GPU capacity:** RunPod **Pods** (REST API launches a
*container* directly — the docker-run pattern with no cloud-init needed),
Nebius, Crusoe, Vast.ai, Vultr, Hyperstack, DigitalOcean GPU Droplets, OVHcloud
all fit the Lambda script pattern. CoreWeave is Kubernetes-native (CKS) — it
belongs to the Helm path, not a VM script.

**Coverage shortcut:** SkyPilot and/or dstack already cover RunPod, Nebius,
Crusoe, Vast.ai, Vultr, DigitalOcean/Paperspace, CoreWeave, Lambda and more —
so one launcher integration reaches nearly all of these without TuFT owning
any per-provider code. Only Beam (Modal-pattern) and RunPod Pods would justify
small dedicated helpers if user demand shows up; Hyperstack/OVHcloud are
covered by neither launcher and are not worth bespoke scripts today.

## 8. Comparison and recommendation

| Option | Roadmap coverage | Fit for TuFT's contract | Code we own | Cost model for users |
|---|---|---|---|---|
| **Helm chart (K8s)** | **AWS + GCP + Azure + Alibaba + on-prem** — the only single artifact covering all four | Excellent — maps to standard primitives; foundation for multi-node (KubeRay), observability (Prometheus/OTel), GPU sharing | 1 chart + thin per-cloud guides | pay for node pool; scale node pool to 0 manually/autoscaler |
| SkyPilot helper | AWS + GCP + Azure + Lambda + ~10 more; **no Alibaba** | Good (VM path); Ray caveat manageable; volumes weak on VMs | 1 YAML + thin wrapper | autostop (idle → stop) |
| dstack helper | similar to SkyPilot; **no Alibaba**; volumes missing on Azure/Lambda | Good (`type: service` maps 1:1) | 1 config | scale-to-zero via gateway |
| 4× per-cloud VM scripts | all four, one at a time | Good | 4 full launch.py scripts + CI + docs | terminate manually |
| PAI-EAS guide | Alibaba only | Good (only genuine managed fit) | ~0 (docs only) | managed, per-instance |
| Managed inference platforms (SageMaker/Vertex/Azure ML endpoints) | — | ❌ structural mismatch — do not pursue | — | — |
| Serverless GPU (Cloud Run, ACA, Function Compute) | — | ❌ GPU class + statelessness | — | — |

**Recommendation: Kubernetes first.** Three reasons:

1. **It is the only move that satisfies the roadmap sentence with one
   integration** — including Alibaba Cloud, which SkyPilot and dstack both
   skip. A Helm chart plus four short "create a GPU node pool on
   EKS/GKE/AKS/ACK, then `helm install`" guides covers everything.
2. **Every other near-term roadmap item lands on it.** Multi-node training
   (KubeRay), observability (Prometheus/Grafana/OTel collectors are
   K8s-native), and multi-tenant GPU sharing (MIG/time-slicing/cGPU) all
   assume a cluster substrate. The chart is the foundation, not a detour.
3. **It matches the stated positioning** — "sits above the infrastructure
   layer (Kubernetes, cloud platforms, GPU clusters)" — and meets enterprise
   users where they already are: existing GPU clusters.

SkyPilot remains the right **second** step: it adds the "I don't have a
cluster, just rent me a VM" path on AWS/GCP/Azure (+ Lambda, RunPod, Nebius, …)
with one YAML, replacing the need for per-cloud VM scripts. Keeping both paths
mirrors today's split: Helm ≈ the infrastructure path, SkyPilot ≈ the
rent-a-GPU path (as Modal/Lambda are today).

## 9. Proposed next step

**Ship `deploy/kubernetes/`: a Helm chart + docs page + E2E validation on one
managed cloud**, following the existing backend conventions.

Suggested PR scope:

```
deploy/kubernetes/
├── README.md                     # quickstart + per-cloud node-pool one-liners
└── chart/                        # helm chart "tuft"
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml       # replicas: 1 enforced, strategy: Recreate,
        │                         #   nvidia.com/gpu resources, /dev/shm emptyDir
        │                         #   (medium: Memory, sizeLimit), startupProbe on
        │                         #   /api/v1/healthz, GPU tolerations/nodeSelector
        ├── config-secret.yaml    # full tuft_config.yaml as a Secret (it embeds
        │                         #   the tml- API keys), or existingSecret ref
        ├── pvc.yaml              # /data (HF cache + checkpoints), RWO
        ├── service.yaml          # ClusterIP default; LoadBalancer optional
        ├── ingress.yaml          # optional
        └── NOTES.txt             # port-forward + Tinker connect snippet
docs/sphinx_doc/source/deployment/kubernetes.md   # same Yoda walkthrough
```

Key `values.yaml` knobs: `image`, `gpuCount`, `shmSize`, `resources`
(memory limit must cover shm + process RSS), `config` (inline tuft_config) /
`existingConfigSecret`, `hfTokenSecret`, `persistence.{size,storageClass}`,
`service.type`, `ingress.*`, `redis.url`, `nodeSelector`/`tolerations`
(GPU taints differ per cloud), `runtimeClassName`.

Acceptance criteria (mirrors `deploy-e2e.yml`):

1. `helm lint` + `helm template` golden test in the normal CI (no cluster
   needed).
2. Manual E2E on one managed cluster: install with `Qwen/Qwen3-0.6B` on an
   L4/A10-class node, `/api/v1/healthz` gates green, `get_server_capabilities`
   answers, Yoda SFT run completes, `helm uninstall` + node pool teardown.
   Later: add a `k8s` option to `deploy-e2e.yml` (needs cluster credentials as
   repo secrets).
3. Docs guide reaches parity with the Modal/Lambda pages.

Which cloud to validate first is a maintainer call: **ACK** serves the core
audience and is the coverage gap everything else misses; **GKE Autopilot** or
**EKS Auto Mode** are the lowest-friction validation targets (drivers + device
plugin fully managed, per-pod GPU provisioning). Validating on one + smoke
docs for the others is enough for a first PR.

Explicit non-goals for v1 (so reviewers don't have to re-litigate): no
operator/CRD (KubeAI/KAITO territory — later, if ever), no SkyServe (beta),
no scale-to-zero on K8s (users who want that have Modal), no managed inference
platforms (§6), no multi-node (KubeRay comes with the distributed-training
milestone — just keep the pod spec reusable as a future head-pod template).

Follow-on sequence after the chart lands:

1. Per-cloud quickstart guides: ACK + EKS, then GKE + AKS (thin, validated).
2. `deploy/sky/` SkyPilot task YAML + helper (covers the VM path on
   AWS/GCP/Azure/Lambda/RunPod/… in one file; document the
   `ray.init(address="auto")` prohibition and `--shm-size` run option).
3. PAI-EAS deployment guide (docs-only; best managed fit, China audience).
4. Opportunistic: Beam / RunPod Pods helpers if user demand shows up.

### Open questions for the RFC discussion

- First validation cloud: ACK (audience, coverage gap) vs GKE/EKS (friction)?
- Publish the chart to `ghcr.io/agentscope-ai` as an OCI artifact alongside
  the image?
- Config secret shape: whole `tuft_config.yaml` as one Secret (simple, matches
  the "one portable file" convention) vs splitting API keys into env vars?
- SkyPilot vs dstack for phase 2 (default lean: SkyPilot — Apache-2.0, larger
  community — despite the internal-Ray caveat; dstack's `service` + `shm_size`
  ergonomics are the counterargument).

