# Research: Cloud-Native Deployment Options for TuFT

*Status: research note / RFC draft — 2026-07-18*

This note surveys deployment options for TuFT beyond the two existing backends
(Modal and Lambda Cloud), targeting the roadmap item:

> **Cloud-native deployment**: Integration with AWS, Alibaba Cloud, GCP, Azure
> and Kubernetes orchestration.

It ends with a concrete recommendation for the next step.

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

<!-- TODO: section 4 — Kubernetes path (agent B) -->

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

<!-- TODO: section 6 — per-cloud native options (agent C) -->
<!-- TODO: section 7 — other GPU clouds (agent D) -->
<!-- TODO: section 8 — comparison + recommendation -->
<!-- TODO: section 9 — proposed next step, PR scope, open questions -->
