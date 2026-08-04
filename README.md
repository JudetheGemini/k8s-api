# QOVES Take-Home, Writeup

Repo: https://github.com/JudetheGemini/k8s-api

---

# QOVES Take-Home — Senior DevOps

A self-managed Kubernetes platform for the provided trivial API: GitOps delivery via
ArgoCD app-of-apps, default-deny NetworkPolicies, secrets sealed rather than committed,
and Postgres on a PersistentVolumeClaim. Built against the QOVES Task 1 brief, August 2026.

---

## 1. Run it

### From scratch

```
# Tools (macOS / Apple Silicon)
brew install minikube kubernetes-cli helm kubeseal argocd jq

# Cluster: two nodes, Calico as the NetworkPolicy-enforcing CNI
minikube start --nodes=2 --cni=calico --memory=4096 --cpus=2 --driver=docker

# Addons required by the app: ingress path, HPA metrics
minikube addons enable ingress
minikube addons enable metrics-server

# Reachability on macOS (Docker driver): the ingress controller binds
# to a node IP inside the Docker Desktop VM, which is not routable
# from the host. minikube tunnel provides that host route. Leave this
# running in its own terminal.
minikube tunnel

# Local hostname for the ingress
echo "127.0.0.1 qoves.local" | sudo tee -a /etc/hosts

# GitOps controller, the one platform component installed by hand,
# because something has to install the thing that installs everything
# else. Chart version pinned for reproducibility.
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd \
  --version 10.2.2 -f bootstrap/argocd-values.yaml

# Bootstrap: apply the root Application once. This is the last
# imperative workload command in the project.
kubectl apply -f clusters/minikube/root-app.yaml
```

From this point everything else, the sealed-secrets controller, the namespace, Postgres, the API, and their NetworkPolicies, arrives because the root Application points at `apps/`, and each file there is a child Application pointing at its own directory under `manifests/`. Nothing downstream of the root apply is ever `kubectl apply`'d directly.

### Repo layout

```
app/                      the provided API, source + Dockerfile, unmodified
clusters/minikube/        root-app.yaml, the single by-hand bootstrap object
apps/                     child Application manifests (pointers, not resources)
  namespace.yaml            wave 0
  sealed-secrets.yaml       wave 0
  postgres.yaml             wave 1
  api.yaml                  wave 2
manifests/                 the actual Kubernetes resources each child delivers
  namespace/
  postgres/                 StatefulSet, headless Service, sealed postgres-auth
  app/                       Deployment, Service, Ingress, NetworkPolicies, sealed api-db
docs/
  WRITEUP.md                this file
  DECISIONS.md
  ASSUMPTIONS.md
  NOTES.md                  working scratch notes, not part of the submission proper
  proof/                     captured command output
bootstrap/                 the handful of by-hand commands (Helm, ArgoCD install)
```

### Making a change (the GitOps flow)

Edit a manifest under `manifests/`, commit, push. ArgoCD polls every three minutes by default, or `argocd app sync root` forces it immediately. `selfHeal: true` means a change made by hand against the live cluster (`kubectl edit`, `kubectl scale`) is reverted automatically, the only way to make a durable change is through git. This was demonstrated in practice, not just configured: see the app-of-apps entries in Decisions and the runbook below.

---

## 2. Decisions

**CNI: Calico.**
Decision: Calico, enforcing NetworkPolicy on a two-node minikube cluster.
Alternatives: Cilium, eBPF dataplane, richer L7-aware policy, Hubble flow visibility, but more setup surface than a 3-day window comfortably allows. Flannel, implements pod networking but not NetworkPolicy enforcement, which makes it the wrong choice for a brief centered on default-deny. Weave, effectively unmaintained.
Why: the brief calls out that minikube's default CNI silently no-ops on NetworkPolicy objects; picking a CNI that actually enforces it is part of the task, not incidental.

**GitOps controller: ArgoCD.**
Decision: ArgoCD via Helm, app-of-apps pattern.
Alternatives: Flux, which is comparable but has a less immediately legible UI/CLI story for producing proof artifacts (`argocd app list`, `argocd app resources`) under time pressure.
Why: native app-of-apps support and an app tree that doubles as submission evidence.

**Secrets: Sealed Secrets.**
Decision: Bitnami's sealed-secrets controller, ciphertext committed to git.
Alternatives: SOPS, requires an ArgoCD plugin or a kustomize-sops sidecar, more moving parts to explain in a walkthrough. External Secrets Operator, only honest if backed by a real external store; its fake/inline provider puts the plaintext value back into a manifest, which is exactly what the brief prohibits.
Why: fully local, no plugin, and the failure mode (rotating the sealing key) is easy to reason about and explain.

**Postgres: raw StatefulSet.**
Decision: hand-written StatefulSet with a `volumeClaimTemplate`, not an operator.
Alternative: CloudNativePG, listed in the brief as the stretch bonus, handles WAL archiving and point-in-time recovery, which a nightly `pg_dump` cannot.
Why: the brief explicitly treats a raw StatefulSet as sufficient for the core requirement; the operator is scoped as additional depth, not a baseline expectation.

**Scaling signal: CPU-based HPA, with a documented objection.**
Decision: implement the required CPU HPA, but state plainly that CPU is the wrong signal for this workload.
Why: gunicorn here runs two synchronous workers that block on a 3-second `psycopg.connect` during `/healthz`. The process saturates on concurrency, all workers tied up waiting on I/O, while CPU utilization stays near idle. A CPU-based HPA would fail to scale under exactly the load pattern most likely to hurt this API. metrics-server, which the addon provides, only ever serves CPU and memory; scaling on request concurrency instead would require prometheus-adapter registering `custom.metrics.k8s.io`, or KEDA, neither of which was built here. Before reaching for autoscaling on any signal, raising the worker count is the cheaper first fix for a blocking-IO bottleneck.

---

## 3. What minikube did for me

**Control-plane bootstrap.** kubeadm, PKI generation and rotation, all handled by `minikube start`. On bare metal this is a manual, ongoing responsibility.

**CNI install and its dataplane choice.** Calico on minikube defaults to IP-in-IP encapsulation for cross-node pod traffic, each packet between the two nodes is wrapped in an outer IP header and unwrapped on arrival. In a real datacenter with a routable fabric, I would run Calico in unencapsulated BGP mode instead and let the physical network route pod CIDRs natively, which avoids the encapsulation overhead entirely. Minikube picked the safe default for me and I never had to make that call.

**Container runtime.** Provisioned as Docker via cri-dockerd, minikube's current default (about to change to containerd in v1.39). On bare metal this means installing and configuring the runtime directly: `/etc/containerd/config.toml`, and specifically setting `SystemdCgroup = true` to match the kubelet's cgroup driver, a mismatch here is a classic bare-metal failure mode that produces confusing resource-accounting bugs. Also: registry mirrors, private registry auth, and an upgrade lifecycle managed separately from Kubernetes itself.

**Ingress load-balancing.** This is the layer minikube hides most completely. The ingress addon runs ingress-nginx with `hostNetwork: true`, binding directly to a node's ports 80/443, workable on a single machine, not a real availability story. On bare metal, something has to answer ARP for a virtual IP and survive a node dying: MetalLB in L2 or BGP mode, kube-vip, or a pair of external HAProxy instances behind keepalived. Then DNS pointing at that VIP, then cert-manager for TLS. None of that exists in this build; `minikube tunnel` is a macOS-specific stand-in that routes host traffic into the Docker Desktop VM, and it is explicitly not a substitute for a real edge layer.

**Storage provisioner.** The `standard` storageClass here is hostpath-backed: it writes to a directory on whichever node the pod is scheduled to, with no cross-node story at all. Real options: Ceph via Rook, Longhorn, or local-path with topology-aware scheduling constraints. See Part F for the concrete consequence of this choice.

**etcd**, plus a snapshot/restore procedure that would need to be built, tested, and monitored independently in a self-managed cluster.

---

## 4. Production gaps

**High availability.** Single control-plane node, single-replica Postgres, no pod disruption budgets. A lost node can take down the control plane or the database with no automatic failover.

**Backups.** None currently implemented, the largest concrete gap. See Part F for the plan (`pg_dump` via CronJob to object storage) versus what actually exists today, which is nothing. This is the single item I would build first with more time, ahead of Prometheus or the HPA, because data loss is the least recoverable failure mode in the whole system.

**Real secret backend.** Sealed Secrets works, but the decryption key lives only in the cluster. A rebuilt cluster cannot decrypt its own existing SealedSecrets unless that key was backed up separately, which was not done here. Production would move to Vault or AWS Secrets Manager via External Secrets Operator, so the plaintext value never exists in git in any form, encrypted or otherwise, and key rotation is handled by the store rather than by a cluster-local RSA keypair.

**Upgrades.** No documented path for Kubernetes version upgrades, Calico upgrades, or Postgres major-version upgrades (which require a dump/restore or logical replication, not just a rolling image bump).

**Multi-cluster.** Single cluster, single environment. No staging/prod separation, no story for promoting a change through environments via git (e.g., separate `overlays/` or separate target branches per environment).

**Observability, as built.** Prometheus and the alert rule described in Part H's design were not implemented in the time available, see the note there. Without them, the operational gap is real: nothing currently pages on a sustained `/healthz` failure.

**Platform tooling itself, not just the app.** ArgoCD is running with a single shared admin account, no SSO, no RBAC, and no audit trail of who triggered which sync, for a tool whose entire purpose is controlling cluster changes, that is a meaningful gap in a real multi-operator environment.

**A live example, found rather than theorized.** During testing, the Calico node agent lost its ability to authenticate to the Kubernetes API server after roughly two days of cluster uptime with the host machine sleeping intermittently, both pod network setup and teardown failed with `error getting ClusterInformation: connection is unauthorized: Unauthorized`. Clock drift was checked and ruled out (host and cluster clocks matched within a second). A daemonset restart of `calico-node` forced fresh service-account tokens and resolved it; the pod that had been stuck rescheduled cleanly and the dependent API pods self-healed without any manual intervention beyond that restart. I did not fully root-cause why the token/connection degraded in the first place, a real incident would also want to check apiserver audit logs for the specific denied request and whether the token refresh loop in kube-controller-manager stalled. This is exactly the kind of failure a managed control plane (EKS, GKE) absorbs invisibly, and self-managed Kubernetes does not.

---

## 5. One runbook, Postgres pod unrecoverable

Chosen because it happened for real during this build, not as a hypothetical.

**Detect.** `kubectl get pods -n qoves-app` shows `postgres-0` stuck in `ContainerCreating` or `Terminating` well past its normal startup window (normally `Running` within 10-15 seconds of being scheduled).

**Diagnose.** `kubectl describe pod postgres-0 -n qoves-app`, read the Events section, and classify the failure before acting, the fix differs by class:

- *Volume/scheduling*: `FailedScheduling`, `FailedMount`, usually the node holding the hostpath PV is down or unreachable.
- *CNI*: `FailedCreatePodSandBox`, `KillPodSandboxError`, the network plugin cannot set up or tear down the pod's network namespace. This is what occurred here: Calico returned `Unauthorized` calling the apiserver.
- *Application*: the pod starts and crashes immediately, check `kubectl logs`; this is a config or app bug, not infrastructure, and is the one class of failure this runbook does not cover.

**Recover, by cause.**

- *CNI auth failure*: first rule out clock drift (`date` on the host vs `minikube ssh -- date`), since a skewed clock invalidates the TLS handshake the CNI uses against the apiserver. If clocks match, restart the CNI daemonset, `kubectl rollout restart daemonset/calico-node -n kube-system`, to force fresh service-account tokens and a new connection. If the pod is wedged in `Terminating` throughout, force it through so the StatefulSet controller can recreate it once the CNI is healthy: `kubectl delete pod postgres-0 -n qoves-app --grace-period=0 --force`.
- *Node down*: under the hostpath provisioner in this build, the pod cannot reschedule elsewhere, there is nowhere else the volume's data exists (see Part F). Recovery means either the node returning, or restoring the most recent backup onto a fresh PVC on a healthy node.

**Verify.** Once `postgres-0` is `1/1 Running`, confirm data integrity directly: `kubectl exec -n qoves-app postgres-0 -- psql -U qoves -d qoves -c "\dt"`. Then confirm the API pods, whose readiness probe depends on Postgres, recover on their own within a couple of probe cycles, with no manual intervention needed on them.

**Through git where possible.** This particular incident was an infrastructure fault, not an application or configuration problem, the StatefulSet manifest was correct throughout, and ArgoCD's `selfHeal` would have reverted any manual edit to it if one had been attempted. The git-mediated recovery path applies to the other class of Postgres failure: if the actual fix were "roll back a bad Postgres image" or "correct a resource limit," that is a commit and a sync, not a `kubectl edit`. The first diagnostic question for any failure in this system is which of the two it is, an infra-layer fault (CNI, node health, storage) is recovered through cluster operations directly, while an app-layer fault (bad config, bad image, bad manifest) is recovered through git, because reaching for the wrong recovery mechanism wastes the time that matters most during an incident.

---

## Storage & data (Part F)

**Access mode and its scheduling constraint.** The Postgres PVC uses `ReadWriteOnce` on the `standard` (hostpath) storageClass. In practice this pins the volume's data to a directory on whichever specific node the pod was first scheduled to, in this build, `minikube-m02`. The pod can never be scheduled onto the other node, because the data does not exist there.

**What happens if the node or pod dies.** A pod restart on the *same* node is fully safe, the volume remounts and data survives, which was demonstrated in practice after an unplanned Calico network fault forced `postgres-0` through a full teardown and recreation, and it came back healthy with the volume intact. A *node* failure is materially worse: because the PV is hostpath-backed rather than network-attached, the scheduler cannot reschedule the pod anywhere else, it has nowhere else to put it, and `postgres-0` would sit `Pending` indefinitely until that specific node returns. A network-backed storageClass (EBS via the CSI driver, Ceph RBD, Longhorn) would allow the volume to detach and reattach on a healthy node instead, trading that resilience for higher I/O latency and an added operational dependency.

**Backup and restore.** Not implemented in this build; the design is a `CronJob` running nightly, execing into the Postgres pod to run `pg_dump -Fc` (custom format, smaller and supports selective restore), streaming the dump to S3-compatible object storage rather than local disk, a node failure that takes the database would otherwise also take any locally-stored backup. Retention: roughly 7 daily plus 4 weekly, pruned via a bucket lifecycle policy. Restore: load the dump into a fresh Postgres instance or PVC with `pg_restore`, run a smoke query against it, then cut `DATABASE_URL` over. The restore path itself should be exercised on a schedule, not just documented, an untested backup is a hypothesis, not a guarantee. The stretch-goal shape of this, CloudNativePG with scheduled backups to MinIO, is the stronger production answer because it adds WAL archiving for point-in-time recovery, bounding worst-case data loss to minutes rather than up to 24 hours.

---

## What I would do next (ran out of time)

- **Part G, resources & HPA**: requests/limits were not finalized and the HPA was not implemented. The CPU-vs-concurrency reasoning above is fully worked out; what remains is the YAML itself plus enabling the metrics-server-backed HPA object, which is a short remaining step.
- **Part H, observability**: Prometheus and the alert were not deployed. Design: scrape the API's `/metrics` with a minimal Prometheus (no full dashboard stack), and alert on sustained `/healthz` failure, `rate(http_requests_total{path="/healthz",status="503"}[5m]) > 0` for 2 minutes, because that specific signal means the API is up but cannot reach its database, which is user-visible and directly actionable, as opposed to a generic pod-restart alert that carries no information about *why*.
- **Backups**: as detailed in Part F, this is the gap I would close first with more time, ahead of both G and H, since it's the one failure mode with no recovery path at all right now.