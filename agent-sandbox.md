## Building an Agent Sandbox from Scratch

Notes on building a sandbox where an AI agent can run untrusted code (shell, git, package installs, tests). The goal is both a working tool and a way to learn the internals.

Key insight: the isolation runtime is the small part. Most of the work is the **control plane** around it.

### What a sandbox has to do

| Concern | What it means |
|---|---|
| Isolation | Agent runs untrusted code; the boundary must hold |
| Exec API | Run a command, stream stdout/stderr, read/write files, enforce timeouts |
| Lifecycle | Create, pause/snapshot, resume, destroy; warm pools for fast start |
| Filesystem | Clone the repo, persist the workspace across sessions, disk quotas |
| Network egress | Default-deny, allowlist proxy, logging |
| Secrets | Sandbox never holds real credentials; a proxy injects them into outbound requests |
| Resource limits | CPU, memory, PIDs, disk |
| Observability | Log every command and connection so a session can be replayed |

### Isolation options (weakest → strongest)

1. **Plain containers (runc)**: namespaces + cgroups + seccomp. Fast, but share the host kernel. OK for your own code, weak for untrusted code.
2. **gVisor (`runsc`)**: user-space kernel that intercepts syscalls. Drop-in runtime for Docker/k8s. Used by GKE Sandbox and Cloud Run.
3. **Kata Containers**: each pod runs in a lightweight VM, but still looks like a container to Kubernetes.
4. **Firecracker microVMs**: KVM-based, ~125ms boot. Powers AWS Lambda, Fargate, and E2B. Needs bare metal or nested virtualization (`/dev/kvm`).
5. **WASM (Wasmtime etc.)**: strong and fast, but can't run arbitrary Linux tooling (git, pip, compilers). Fine for "run this snippet", not for coding agents.

### Stack options

#### A. Single host, DIY (best for learning)
- Docker or Firecracker on one Linux box.
- Small control-plane service (Go or Python) exposing `create` / `exec` / `destroy`.
- Firecracker needs KVM: bare metal, a home server, or a cloud VM with nested virt.
- You learn everything: rootfs images, tap networking, the jailer, snapshots.

#### B. Kubernetes-native
- One pod per sandbox; `RuntimeClass` set to gVisor or Kata for isolation.
- A `Sandbox` CRD + controller (kubebuilder) handles warm pools, idle timeouts, and reclaiming.
- Egress: NetworkPolicy or Cilium FQDN policies, routed through a proxy pod that injects credentials.
- Storage: PVC per workspace, or ephemeral storage + push to git.
- Reference: `kubernetes-sigs/agent-sandbox` for a CRD design.
- Pros: scheduling, bin-packing, multi-tenancy for free. Cons: pod startup is seconds without warm pools; k8s itself is heavy to run.

#### C. AWS-native
- **Fargate / ECS tasks**: Firecracker isolation, managed for you. Slow start (tens of seconds), no snapshots.
- **Lambda**: fast, but 15-minute limit, read-only filesystem except `/tmp`, no persistent shell. One-off code execution only.
- **EC2 metal (e.g. `c7i.metal`) + Firecracker yourself**: what E2B-style platforms do. Full control and fast, but you build the scheduler.
- **EKS + Kata or gVisor**: option B on AWS.
- Glue: VPC + security groups, NAT + egress proxy, Secrets Manager, S3 for snapshots/workspaces.
- Managed shortcut: Bedrock AgentCore (code interpreter / runtime sandboxes).

#### D. GCP and others
- Cloud Run (gVisor) and GKE Sandbox: easiest strong isolation.
- Fly Machines: Firecracker VMs behind an API; nice middle ground.

#### E. Buy instead of build (to compare against)
- E2B (open source, Firecracker-based; worth reading the code), Modal, Daytona.

### Learning path

1. **Docker on a laptop.** Build the exec API and a minimal agent loop: model calls a `bash` tool → tool runs `docker exec`. Add timeouts, output truncation, resource limits.
2. **Swap the isolation layer.** Change the runtime to `runsc` (gVisor) with no other code changes. Then try Firecracker on a KVM host: build a rootfs, set up tap networking.
3. **Egress proxy.** Default-deny networking; route traffic through an allowlist proxy (mitmproxy or Envoy) that adds API keys so the sandbox never sees them. This is the most security-relevant piece.
4. **Make it fast.** Warm pools + Firecracker snapshot/restore. Target: < 1s to a ready sandbox.
5. **Orchestrate.** Either a custom scheduler across a few metal hosts (teaches systems) or a Kubernetes controller (teaches the platform).

### Recommendation

- **To learn:** single host + Firecracker (option A). It teaches the fundamentals the managed options hide.
- **To get something useful sooner:** Kubernetes + gVisor + your own controller. On AWS, that's EKS, or Firecracker on a metal instance.
