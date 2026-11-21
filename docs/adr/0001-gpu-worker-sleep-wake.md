# ADR 0001: GPU worker sleep and wake on demand

- **Status:** Proposed
- **Date:** 2026-11-21
- **Issue:** #61 (follows #57; host side in #60; cluster side in
  jordanthomas761/homelab-infra#93)

## Context

`gpu.k8s.internal` is Proxmox VM 201: Ubuntu 24.04, amd64.
It joined the cluster on 2026-11-21 and is tainted `dedicated=gpu:NoSchedule`
from registration. It is meant to be off when idle and start when work
arrives. Today it is always on, and nothing implements sleep or wake.

What the live cluster looks like today (read-only checks, 2026-11-21):

- **Pods on the node:** only DaemonSet pods: `cilium`, `cilium-envoy`,
  `kube-proxy` (all tolerate `operator: Exists`) and
  `kube-prometheus-stack-prometheus-node-exporter` (tolerates every
  `NoSchedule`). Longhorn, MetalLB's speaker and `nginx-gateway-nginx` do not
  run there.
- **Cluster add-ons:** no KEDA and no autoscaler are installed. Sealed
  Secrets is (`kube-system/sealed-secrets`). `llm-reviewer`, the first
  expected caller, has a `default-deny` NetworkPolicy.
- **kube-controller-manager** runs with no `--node-monitor-grace-period`
  flag, so it uses the default of 50s
  ([kube-controller-manager reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)).
- **Cilium** runs in `routing-mode=tunnel`, `tunnel-protocol=vxlan`,
  `kube-proxy-replacement=false`, health checking on, and
  `set-cilium-node-taints=true`.
- **The VM:** `qemu-guest-agent` is active, so `qm shutdown` and the API
  shutdown call go through the agent. Userspace boot takes about 4s with no
  GPU yet. The kubelet has `shutdownGracePeriod: 0s`, so graceful node
  shutdown is **off**.
- **Alertmanager** sends every `warning` and `critical` alert to ntfy.
  `severity=info`, `Watchdog` and `InfoInhibitor` go to the `null` receiver
  (`apps/kube-prometheus-stack/values.yaml` in homelab-infra).

## Decision summary

| Question | Decision |
|---|---|
| Scaling the inference server | KEDA plus the KEDA HTTP add-on scale the inference Deployment between 0 and 1 and hold requests during a cold start. |
| What wakes the VM | A small in-cluster controller, `gpu-power`. It starts VM 201 through the Proxmox API when a pod that needs the GPU node is Pending. |
| What puts it to sleep | The same controller. After the node has had no non-DaemonSet pods for an idle period, it cordons, drains, re-checks, then calls the API's `shutdown` (never `stop`). |
| Proxmox credentials | A privilege-separated API token for a dedicated `pve` user, with `VM.PowerMgmt` and `VM.Audit` on `/vms/201` only, held in a SealedSecret that only `gpu-power` mounts. |
| Node while the VM is off | Keep the Node. It stays cordoned and labelled `asleep` while off. No delete and re-join. |
| First request after a wake | The interceptor holds it, with a readiness timeout of about 5 minutes, then returns `504`. Callers set a client timeout above that, or retry on 5xx. |
| Alerting | Drop the sleeping node's scrape targets with a node-label relabel, and replace three kube-state-metrics rules with copies that exclude an asleep node. Add alerts for failed sleeps and slow wakes. No silences, and no cluster-wide inhibit. |

The sections below give the reasoning and the trade-offs.

## 1. What wakes the VM

**Decision:** a single-replica `gpu-power` controller that runs on the Pis
(never on the GPU node). It watches pods and wakes the VM when **any**
unschedulable Pending pod tolerates `dedicated=gpu` or requests
`nvidia.com/gpu`, and the VM is not running.

That signal covers every source of work in one place:

- **HTTP requests to the inference server.** The KEDA HTTP add-on scales the
  Deployment from 0 to 1. The new pod is Pending because the node is cordoned
  and asleep, and that wakes the VM.
- **Batch Jobs** (fine-tunes, embeddings) that tolerate the taint wake it the
  same way, with no KEDA object needed.
- **Unplanned loss of the VM.** The inference pod is evicted after 300s (see
  section 3), its replacement is Pending, and that triggers a start.

Waking means:

1. Set `power.homelab.internal/state=waking` on the Node.
2. Call `POST /api2/json/nodes/{pve-node}/qemu/201/status/start`.
3. Wait until the Node is `Ready`, the Cilium agent pod on the node is
   `Ready`, and `nvidia.com/gpu` is allocatable once the GPU Operator is in.
4. Uncordon the node and set `state=awake`.

Gating the uncordon on the Cilium pod means nothing schedules onto the node
before its networking is up, without relying on how
`set-cilium-node-taints` treats a node that already exists.

### Alternatives considered

- **KEDA alone, with a scaler that calls Proxmox.** KEDA scalers report
  metrics; they do not take actions. A scaler that starts a VM as a side
  effect of being polled would be an abuse of the interface. It would also be
  polled at KEDA's interval, and could not handle sleep or the
  drain-then-check sequence. KEDA is still used, but only for what it is
  built for (section 4).
- **Cluster Autoscaler, or Karpenter with a Proxmox provider.** These create
  and delete *nodes* from node groups. The model here is "power a fixed,
  already-joined VM on and off". The Proxmox providers are third-party, and
  this is a lot of machinery for one VM.
- **An external trigger** (callers call Proxmox or send a Wake-on-LAN
  packet). This couples every caller to Proxmox, puts a power credential
  next to each caller, and does not cover Jobs or recovery.

**Trade-off:** `gpu-power` is code we write and maintain (an estimated few
hundred lines, plus an image). In return there is one place that decides
power, one credential, and one state machine to reason about.

## 2. What puts it to sleep

**Decision:** `gpu-power` runs this sequence, keeping its state in the Node's
`power.homelab.internal/state` label so a controller restart resumes
correctly:

1. **Idle check.** No non-DaemonSet pod that is not `Succeeded` or `Failed`
   has been on the node for `idleGrace` (default 10 min). KEDA has already
   scaled the inference Deployment to 0 after its `cooldownPeriod` (default
   15 min). The node therefore sleeps at the earliest about 25 min after the
   last request. Skip the whole sequence if the Node has the annotation
   `power.homelab.internal/hold=awake`, the manual override for maintenance
   (for example while #60 is being worked).
2. Set `state=draining` and **cordon** the node.
3. **Drain** through the Eviction API, so PodDisruptionBudgets are honoured.
   Normally there is nothing to evict.
4. **Verify.** Re-list the pods on the node. Abort if any non-DaemonSet pod
   remains, or if any GPU pod is now Pending, because a request arrived during
   the drain. To abort, uncordon, set `state=awake`, and emit an Event and a
   metric.
5. **Shut down.** Call
   `POST /api2/json/nodes/{pve-node}/qemu/201/status/shutdown` with `timeout`
   set (for example 180s) and **never** `forceStop`. Never call
   `status/stop`.
6. **Confirm.** Poll `GET .../status/current` until it reports `stopped`. If
   it is still running after the timeout, set `state=awake`, uncordon, and let
   `GPUWorkerSleepFailed` fire. The controller does not escalate to a hard
   stop; a person decides that.
7. Set `state=asleep`. The node **stays cordoned**, which matters for
   alerting (section 5).

If a GPU pod goes Pending while the node is `draining` or shutting down, the
controller lets the shutdown finish, then starts the VM again. That costs a
reboot, not a lost request: the interceptor holds the request.

**Backstop inside the VM:** enable kubelet graceful node shutdown on the
`gpu` group (for example `shutdownGracePeriod: 60s`,
`shutdownGracePeriodCriticalPods: 20s`). Someone pressing Shutdown in the
Proxmox UI then still gets pods terminated cleanly, and the Node is marked
`NotReady` with the reason "node is shutting down"
([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)).
The kubelet holds a systemd inhibitor lock to do this. logind's
`InhibitDelayMaxSec` must be at least the grace period, and the Proxmox
shutdown `timeout` must be longer than it.

### Proxmox API token

The API schema
([apidoc.js](https://pve.proxmox.com/pve-docs/api-viewer/apidoc.js)) gives
these permission checks:

| Call | Permission |
|---|---|
| `POST .../qemu/{vmid}/status/start` | `VM.PowerMgmt` on `/vms/{vmid}` |
| `POST .../qemu/{vmid}/status/shutdown` | `VM.PowerMgmt` on `/vms/{vmid}` |
| `GET .../qemu/{vmid}/status/current` | `VM.Audit` on `/vms/{vmid}` |

So the token needs `VM.PowerMgmt` and `VM.Audit` on `/vms/201`, and nothing
else. A privilege-separated token gets the **intersection** of its user's
permissions and its own ACLs
([User Management](https://pve.proxmox.com/pve-docs/chapter-pveum.html)), so
grant the role to both on `/vms/201` only. A sketch, to check on the host:

```bash
pveum role add K8sGpuPower --privs "VM.Audit,VM.PowerMgmt"
pveum user add gpu-power@pve --comment "k8s gpu-power controller (VM 201 power only)"
pveum acl modify /vms/201 --users gpu-power@pve --roles K8sGpuPower
pveum user token add gpu-power@pve controller --privsep 1
pveum acl modify /vms/201 --tokens 'gpu-power@pve!controller' --roles K8sGpuPower
pveum user permissions 'gpu-power@pve!controller'   # expect only /vms/201
```

- **Where it lives:** a SealedSecret in homelab-infra, in a `gpu-power`
  namespace, mounted only by the controller. It is sent as
  `Authorization: PVEAPIToken=gpu-power@pve!controller=<uuid>`. Pin the
  Proxmox CA (`/etc/pve/pve-root-ca.pem`) instead of disabling TLS
  verification.
- **Blast radius:** `VM.PowerMgmt` cannot be split. It also allows `stop`,
  `reset` and `suspend`. A leaked token can therefore cause a GPU outage, but
  it cannot read the VM's disk or console, or change its config. The
  controller code never calls those endpoints. This matches the `vm-ctl`
  approach for the control-plane VM: the worst case is an outage, not data
  access.
- **Network:** the controller pod on a Pi must reach the Proxmox host on
  port 8006. This has not been checked yet; the Proxmox host is not in this
  repo's inventory.

### Kubernetes RBAC for `gpu-power`

- `nodes`: `get`, `list`, `watch`, plus `patch` limited with
  `resourceNames: [gpu.k8s.internal]` (for cordoning and the label).
- `pods`: `list`, `watch`.
- `pods/eviction`: `create`.
- `events`: `create`.

Eviction cannot be limited to one node through RBAC. The controller only
evicts pods whose `spec.nodeName` is the GPU node.

## 3. The Node while the VM is off

**Decision: keep the Node.** Leave it cordoned and labelled
`power.homelab.internal/state=asleep`, and accept that it is `NotReady`
while off.

What happens, checked against the live cluster and the upstream docs:

- **After 50s without a heartbeat,** the node lifecycle controller sets
  `Ready=Unknown` and adds `node.kubernetes.io/unreachable` as both
  `NoSchedule` and `NoExecute`.
- **Ordinary pods** get an automatic toleration for `unreachable` and
  `not-ready` with `tolerationSeconds: 300`, so they are evicted 5 minutes
  later
  ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)).
  After a planned sleep there are none, because section 2 leaves only
  DaemonSet pods.
- **DaemonSet pods** automatically tolerate `unreachable` and `not-ready`
  `NoExecute` with no time limit, as well as `unschedulable`
  ([DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)).
  `cilium`, `cilium-envoy`, `kube-proxy` and `node-exporter` therefore stay
  bound to the node, shown as not ready. They are not evicted and not
  rescheduled, and they resume when the VM boots. Cordoning does not remove
  them. The GPU Operator's operand DaemonSets will behave the same way.
- **Cilium:** the `CiliumNode` object stays. The other agents keep the node's
  VXLAN tunnel entry and report it unreachable in `cilium-health`. Nothing
  scrapes Cilium's metrics today, so this does not alert. Services lose
  nothing, because a pod on a NotReady node is marked not ready and dropped
  from its Service endpoints.
- **The inference Deployment and `unreachable` eviction:**
  - *Planned sleep.* KEDA has already scaled it to 0, so nothing is evicted.
  - *Unplanned loss* (host crash, power cut). The pod is evicted after 300s.
    It stays `Terminating` until the kubelet returns, but a Deployment
    creates its replacement straight away. The replacement is Pending, which
    wakes the VM (section 1). Use a **Deployment, not a StatefulSet**: a
    StatefulSet waits for the old pod to be confirmed gone, so it would sit
    stuck until someone wakes the VM by hand.
  - **`strategy: Recreate`.** There is one GPU, so a rolling update would
    wait forever for a second `nvidia.com/gpu`.
  - The local-path PV for model weights pins the pod to this node through
    node affinity. That is fine: there is only one GPU node.
  - Lowering the pod's `unreachable` `tolerationSeconds` would only start
    recovery sooner after a crash. It is optional.
- **A crash while awake:** the node is uncordoned, so the existing
  `KubeNodeNotReady` alert fires as it should. If the VM reports `running`
  but the Node has been unreachable for more than 10 minutes, the controller
  alerts and does **not** reset the VM.

### Why not delete the Node and re-join

The issue assumed this needs a new bootstrap token and `kubeadm join` on
every wake. It does not. The kubelet keeps its client certificate, and on
start it re-creates its own Node object, applying `--register-with-taints`.
It is still the worse option:

- **It loses everything set through the API:** `node.longhorn.io/create-default-disk=false`,
  `node-role.kubernetes.io/gpu` (#60; the kubelet may not set that label
  itself), the GPU Operator's NFD labels, and any annotations. The controller
  would have to re-apply them on every wake.
- **It races with the kubelet:** delete the Node while the kubelet is still
  running and the kubelet immediately re-creates it. The delete has to wait
  until Proxmox reports `stopped`, and a controller crash in that window
  leaves the "kept" state anyway.
- **It churns everything keyed to the Node:** the `CiliumNode` and pod CIDR
  allocation, and the node's identity in Grafana, Headlamp and
  kube-state-metrics history.

What deleting would buy is cleaner alerts, and section 5 gets those without
deleting.

### Long sleeps

The kubelet rotates its client certificate (kubeadm issues 1-year
certificates) only while it runs. Unattended upgrades and DaemonSet image
rollouts also wait for the VM to be up. **`gpu-power` wakes the VM on a
schedule** (for example on the first Sunday of each month, for 2 hours) and
otherwise treats that wake like any other. Note that
`KubeletClientCertificateExpiration` is only evaluated while the kubelet is
scraped.

## 4. The first request after a wake

**Decision:** the KEDA HTTP add-on interceptor (v0.16) sits in front of the
inference Service and **holds requests** until a pod is ready
([Configure Cold-Start Behavior](https://keda.sh/http-add-on/0.16/user-guide/configure-cold-start/)).
Set a readiness timeout of about 300s on the route
(`InterceptorRoute.spec.timeouts.readiness`, or `KEDA_HTTP_READINESS_TIMEOUT`).
When it expires, the caller gets **`504 Gateway Timeout`**
([Configure Timeouts](https://keda.sh/http-add-on/0.16/user-guide/configure-timeouts/)).
The interceptor returns `503` when more requests are waiting than
`coldStart.maxPendingRequests` allows. Responses that involved a cold start
carry the `X-KEDA-HTTP-Cold-Start` header.

Use the add-on's current `InterceptorRoute` CRD (`http.keda.sh/v1beta1`).
`HTTPScaledObject` is deprecated in 0.16
([InterceptorRoute](https://keda.sh/http-add-on/0.16/reference/interceptorroute/)).
Scale-down after idle is the KEDA ScaledObject's `cooldownPeriod`, not a
route field
([Scaling](https://keda.sh/http-add-on/0.16/concepts/scaling/)).

### Expected cold start (to be measured)

| Stage | Estimate | Notes |
|---|---|---|
| VM start, OVMF, boot | 20–40s | 4s userspace measured today; passthrough and OVMF add time |
| kubelet `Ready`, Cilium agent `Ready` | 15–30s | |
| `nvidia.com/gpu` allocatable | seconds | PR #64 installs the driver on the host (DKMS, pinned), so a wake does not wait for the GPU Operator's driver container to rebuild or reload it |
| Image start | seconds | Images stay in containerd's store on the VM's disk across shutdowns |
| Model into VRAM | 10s–2 min | Weights are read from `/var/lib/models` (PR #64's second disk). llama.cpp and a GGUF file are fast; vLLM's startup is slower |

The estimated total is **1.5 to 4 minutes**, which is why the timeout is
300s and not KEDA's examples' 30s.

### What callers do

- **The `llm-reviewer`** (homelab-infra #71) and other API clients set a
  request timeout above 300s, or retry on `504` and `503`. Many
  OpenAI-compatible SDKs already retry 5xx; check the reviewer's client.
- **Long generations** that are not streamed can hit the interceptor's
  300s response-header timeout. Stream, or raise `timeouts.responseHeader`
  on the route.
- **The `llm-reviewer`'s `default-deny` policy** needs an egress rule to the
  interceptor.
- **Option, not the default:** `coldStart.fallback.service` could send
  cold-start traffic to the Mac Mini's `mlx-openai-server` instead of
  waiting. It serves a different model, so it is only worth it for callers
  that do not care which model answers.

### Alternatives considered

- **503 with `Retry-After` straight away.** This is simpler, but it pushes
  retry logic into every caller and gives a bad first experience for
  interactive use. The interceptor gives both: a bounded wait, then a
  5xx.
- **`gpu-power` as the front-door proxy** (one component instead of KEDA plus
  the add-on). This is less to install, but it means more code of our own on
  the request path (streaming, timeouts, back-pressure). Keep it as the
  fallback plan if the add-on turns out to be a problem on the Pis. Check
  that the add-on's interceptor and scaler images are published for arm64.

## 5. Alerting

These are the live `PrometheusRule`s from kube-prometheus-stack (read-only,
2026-11-21). With the node **cordoned while asleep**, this is what a planned
sleep would set off. Every row marked "Fires" is a `warning`, which today
sends an ntfy push.

| Alert | `for` | During planned sleep | Why |
|---|---|---|---|
| `KubeNodeNotReady` | 15m | Silent | Rule requires `kube_node_spec_unschedulable == 0`; the node is cordoned |
| `KubeNodeReadinessFlapping` | 15m | Silent | Same exclusion |
| `KubeNodeUnreachable` | 15m | **Fires** | Keys on the `unreachable` taint, with no cordon exclusion |
| `TargetDown` (`node-exporter`) | 10m | **Fires** | 1 of 4 targets down is 25%, above the 10% threshold |
| `TargetDown` (`kubelet`, `kube-proxy`) | 10m | **Fires**, expected | Prometheus endpoints discovery keeps not-ready addresses (`__meta_kubernetes_endpoint_ready=false`), and prometheus-operator appears to keep NotReady nodes in the kubelet Endpoints ([prometheus-operator#6548](https://github.com/prometheus-operator/prometheus-operator/issues/6548)). To confirm on the first sleep test. |
| `KubeletInstanceUnreachable`, `KubeProxyInstanceUnreachable` | 15m | **Fires**, expected | `up == 0` for the same targets |
| `KubeDaemonSetRolloutStuck` | 15m | **Fires** | 4 DaemonSets (later also the GPU Operator's) report `number_available` 3, against `desired` 4 |
| `KubePodNotReady` | 15m | **Fires** | One per DaemonSet pod on the node, and the owner is not a Job |
| `KubeletDown` | 15m | Silent | Only fires when no kubelet at all is up |

### Decision

Fix this at the source. Do not use silences or an Alertmanager inhibit.

1. **Mark the state on the Node.** `gpu-power` sets
   `power.homelab.internal/state`. Add that label to kube-state-metrics'
   `metricLabelsAllowlist` for nodes, so
   `kube_node_labels{label_power_homelab_internal_state="asleep"}` exists.
2. **Drop the sleeping node's scrape targets.** On the `node-exporter`,
   `kubelet` and `kube-proxy` ServiceMonitors, set `attachMetadata.node: true`
   and add a relabel that drops targets with
   `__meta_kubernetes_node_label_power_homelab_internal_state="asleep"`.
   `attach_metadata` works for the pod, endpoints and endpointslice
   discovery roles
   ([Prometheus `kubernetes_sd_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config),
   [ServiceMonitor `AttachMetadata`](https://prometheus-operator.dev/docs/api-reference/api/)).
   The targets disappear instead of reporting `up == 0`. That removes the
   `TargetDown` and `*InstanceUnreachable` rows, with no rule changes. Use the
   chart's values where it exposes them; otherwise this needs extra
   ServiceMonitors.
3. **Replace the three rules that come from kube-state-metrics.** List
   `KubeNodeUnreachable`, `KubeDaemonSetRolloutStuck` and `KubePodNotReady` in
   the chart's `defaultRules.disabled`, and add copies that exclude the
   asleep node. Sketches, to validate in Prometheus before switching over:

   ```promql
   # KubeNodeUnreachable, minus asleep nodes
   (<upstream expr>)
     unless on (node) kube_node_labels{label_power_homelab_internal_state="asleep"}

   # KubePodNotReady, minus pods bound to an asleep node
   (<upstream expr>)
     unless on (namespace, pod)
       (kube_pod_info * on (node) group_left() kube_node_labels{label_power_homelab_internal_state="asleep"})

   # KubeDaemonSetRolloutStuck: count the asleep node's DaemonSet pods as available
   #   number_available + (DaemonSet pods on an asleep node, or 0) != desired
   ```
4. **Add alerts for the feature itself:**
   - `GPUWorkerAsleep`: `info`, so it is visible in the UIs and goes to the
     `null` receiver.
   - `GPUWorkerSleepFailed`: `warning`. A drain or shutdown aborted.
   - `GPUWorkerWakeSlow`: `warning`. A GPU pod has been Pending for more than
     10 min.
   - `GPUWorkerPowerControllerDown`: `warning`. The controller's metrics are
     absent.

   An unplanned outage still shows up: an awake node is uncordoned, so
   `KubeNodeNotReady` fires.

**Why not an Alertmanager inhibit rule?** An inhibit rule needs a label the
source and target alerts share (`equal`). `TargetDown` and
`KubeDaemonSetRolloutStuck` carry no `node` label. An inhibit rule would
therefore have to mute them for the **whole cluster** whenever the GPU node
sleeps, which is most of the time. A Pi's node-exporter going down would then
go unnoticed.

**Why not silences?** Silences are time-bound, need a write credential for
Alertmanager in the controller, and a forgotten one hides a real failure.
They are still fine for a person doing manual maintenance.

## Consequences

- **New components:** KEDA and its HTTP add-on, as Argo CD apps, and the
  `gpu-power` controller.
- **One new credential:** the Proxmox token, limited to powering VM 201.
- **The inference Deployment's replica count belongs to KEDA.** Leave
  `replicas` out of its manifest, or set Argo CD `ignoreDifferences` on it.
- **Every GPU workload stays tolerant of a cold start,** and none may
  assume the node is up. That was already the rule for storage.
- **Manual work on the VM** sets the `power.homelab.internal/hold=awake`
  annotation first, and removes it afterwards.

## Implementation follow-ups

#61 is design only. The implementation is tracked in:

1. #65: create the Proxmox role, user and token, check that the cluster can
   reach the Proxmox API on port 8006, and enable kubelet graceful node
   shutdown on the `gpu` group.
2. jordanthomas761/homelab-infra#94: KEDA and the HTTP add-on as Argo CD
   apps, plus an `InterceptorRoute` and ScaledObject for the inference
   Deployment.
3. jordanthomas761/homelab-infra#95: the `gpu-power` controller: code,
   multi-arch image, Deployment, RBAC, and the SealedSecret.
4. jordanthomas761/homelab-infra#96: the alerting changes in section 5.
5. #66: measure the cold start end to end, and set the readiness timeout and
   `cooldownPeriod` from the numbers.
