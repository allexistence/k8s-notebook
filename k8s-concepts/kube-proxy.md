# Kubernetes Service Networking: One Request, Three Platforms

How a pod reaches a Service, explained by following a single packet. Covers kube-proxy, `br_netfilter`, `nf_conntrack` and the CNI, first on **RKE2**, then how the same flow differs on **VKS** and **EKS**.

---

## The Setup

```
frontend pod (10.42.0.5, node-1)  ──►  Service "backend" 10.43.0.50:80  ──►  backend pod (10.42.1.7:8080, node-2)
```

Four players, each with one job:

| Player | Job in one line |
|---|---|
| **kube-proxy** | Writes the rules that map a Service IP to real pod IPs |
| **`br_netfilter`** | Makes sure bridged traffic passes through those rules |
| **`nf_conntrack`** | Remembers each translation, keeps connections sticky, reverses NAT on replies |
| **CNI** | Gives pods IPs and carries pod-to-pod traffic, same node and across nodes |

Remember this split: **kube-proxy handles Services. The CNI handles pods.** Pod-to-Service traffic is simply kube-proxy's rewrite followed by ordinary pod-to-pod delivery.

---

## Part 1: Following the Packet (RKE2)

### Step 0: Before any packet moves, kube-proxy sets the stage

In RKE2, kube-proxy runs as a **static pod on every node**. The kubelet starts it straight from a file on disk:

```
/var/lib/rancher/rke2/agent/pod-manifests/kube-proxy.yaml
```

kube-proxy **never touches packets**. It watches Services and EndpointSlices in the API server, then writes iptables rules on its own node that say:

> "Anything going to `10.43.0.50:80` → rewrite it to one of these ready pod IPs."

kube-proxy is the *rule writer*. The **Linux kernel** does all the real work.

### Step 1: The name becomes a virtual IP

`frontend` asks CoreDNS for `backend` and gets back `10.43.0.50`.

That ClusterIP is **virtual**. No interface owns it, nothing listens on it, and it won't answer ping. It exists only as a match condition inside kube-proxy's rules.

Note that CoreDNS is itself reached through a ClusterIP (`10.43.0.10`). So **DNS already depends on kube-proxy** before your app sends a single byte.

### Step 2: The kernel catches the packet

`frontend` sends a packet to `10.43.0.50:80`. It leaves the pod through its veth interface and enters the node's netfilter (iptables) stack.

This is where **`br_netfilter`** matters. If the CNI connects pods through a Linux bridge, bridged traffic normally skips iptables entirely. `br_netfilter` forces it through, so kube-proxy's rules can't be bypassed.

RKE2's default CNI, Canal, *routes* pod traffic instead of bridging it, so iptables sees it anyway. The module is still loaded and expected.

### Step 3: kube-proxy's rules rewrite the destination

The packet walks the chains kube-proxy wrote:

```
KUBE-SERVICES ──match 10.43.0.50:80──► KUBE-SVC-xxx ──random pick──► KUBE-SEP-yyy ──DNAT──► 10.42.1.7:8080
```

The virtual IP is now a real pod IP. **Load balancing happened right here**, as a random pick across the ready endpoints.

### Step 4: conntrack remembers the rewrite

**`nf_conntrack`** records: *"10.42.0.5 → 10.43.0.50 was actually sent to 10.42.1.7."*

That single entry does two jobs:

1. **Stickiness.** Only the *first* packet of a connection goes through the random pick. Every later packet follows the conntrack entry, so the connection stays on the same pod.
2. **Reply translation.** It's needed in Step 6.

### Step 5: The CNI carries the packet across nodes

The destination is now a real pod IP on node-2. **kube-proxy's part is done**; the CNI takes over. In RKE2 the default CNI is **Canal** (Calico + Flannel). It does four things:

1. **Assigns pod IPs (IPAM).** Each node gets a slice of the pod CIDR, for example node-2 owns `10.42.1.0/24`.
2. **Wires the pod to the node.** It creates a veth pair (one end is `eth0` inside the pod, the other is a `cali*` interface on the host) plus a host route per pod.
3. **Connects nodes.** Flannel wraps the packet in **VXLAN (UDP 8472)** addressed to node-2's real IP. Node-2 unwraps it and routes it into the backend pod.
4. **Enforces NetworkPolicy.** Calico (Felix) checks policy along the way.

### Step 6: The reply gets un-NATed

`backend` replies **from `10.42.1.7`**. But `frontend` called `10.43.0.50`, so a reply from an unknown IP would be dropped.

conntrack on node-1 finds its entry from Step 4 and rewrites the reply's source back to `10.43.0.50`. `frontend` accepts it. **Round trip complete.**

### The whole chain at a glance

```
DNS ──► ClusterIP ──► br_netfilter ──► kube-proxy rules (DNAT) ──► conntrack ──► CNI (VXLAN) ──► pod
                                                                       │
                                        reply ◄── conntrack un-NAT ◄───┘
```

---

## Part 2: When a Link Breaks

Walk the same chain to find where it broke:

| Broken link | What you see |
|---|---|
| **CoreDNS or its Service** | Names fail everywhere, but ClusterIPs still work |
| **kube-proxy down** | *Not* an instant outage. Old rules stay in the kernel, so existing Services keep working. But the rules **freeze**: new Services are unreachable and dead pods stay in rotation, causing **intermittent timeouts** |
| **No endpoints** | Instant `connection refused` (kube-proxy installs a REJECT rule) |
| **`br_netfilter` missing** (bridge-based CNI) | Same-node pod → Service hangs; cross-node works |
| **conntrack table full** | Random drops on busy nodes; `dmesg` shows `table full` |
| **Stale UDP conntrack** | DNS/UDP fails right after a pod restarts |
| **CNI / overlay** | Pod IP unreachable cross-node: UDP 8472 blocked, MTU mismatch, or `ip_forward=0` |
| **NetworkPolicy** | Specific flows blocked while everything else works |

### One-minute triage

```
Name fails, ClusterIP works      →  DNS
ClusterIP fails, pod IP works    →  kube-proxy / endpoints / conntrack
Pod IP fails                     →  CNI
```

---

## Part 3: The Same Flow on VKS and EKS

The flow from Part 1 is identical everywhere: **kube-proxy writes rules, conntrack remembers, the CNI delivers.** What changes is *how kube-proxy is run* and *what the CNI does in Step 5*.

### How kube-proxy runs

| | **RKE2** | **VKS** (vSphere Kubernetes Service) | **EKS** |
|---|---|---|---|
| Runs as | **Static pod** `kube-proxy-<node>` | **DaemonSet** `kube-system/kube-proxy` | **DaemonSet** `kube-system/kube-proxy` |
| Deployed by | RKE2 writes the manifest; kubelet starts it | kubeadm at cluster bootstrap | AWS, as the **kube-proxy EKS add-on** |
| Configured via | `kube-proxy-arg` in `/etc/rancher/rke2/config.yaml` | kube-proxy ConfigMap / cluster spec | Add-on configuration |
| Default mode | iptables | iptables | iptables (IPVS, nftables optional) |
| Upgraded | Along with RKE2 | Along with the cluster (TKR) upgrade | **Separately**: bump the add-on after each control-plane upgrade |

### Static pod vs DaemonSet: why it matters

- A **static pod** needs nothing but the node. The kubelet starts it from a file, even if the API server is down. You can't `kubectl edit` it; change it through RKE2's config instead.
- A **DaemonSet** needs the control plane to schedule it. It's easier to manage centrally, but one bad edit rolls out to **every node at once** and breaks Services cluster-wide.

### What changes in Step 5 (the CNI)

| | **RKE2** | **VKS** | **EKS** |
|---|---|---|---|
| Default CNI | Canal (Calico + Flannel) | Antrea (Open vSwitch); Calico optional | **VPC CNI** |
| Pod IPs | From a private pod CIDR | From a private pod CIDR | **Real VPC IPs** from ENIs |
| Cross-node transport | VXLAN overlay, UDP 8472 | Antrea tunnel (Geneve by default) | **No overlay**: VPC routes natively |
| Twist to know | — | **AntreaProxy** handles pod → ClusterIP inside OVS flows, so `iptables-save` doesn't show the full picture | Cross-node failures usually mean **security groups or IP exhaustion**, not an overlay port |

### EKS special cases

- **EKS Auto Mode:** AWS runs networking components, including kube-proxy, as part of the managed node. There's no kube-proxy DaemonSet to manage.
- **Fargate:** no visible kube-proxy pod; AWS handles it inside the runtime.
- **Version skew:** on standard EKS, forgetting to update the kube-proxy add-on after a control-plane upgrade is a common post-upgrade surprise.

### Going kube-proxy-free

All three can replace kube-proxy with **Cilium's eBPF** Service handling. eBPF then does the Step 3 rewrite and its own connection tracking, and kube-proxy disappears:

- RKE2: `disable-kube-proxy: true`
- EKS: remove the kube-proxy add-on

---

## Part 4: Commands, in the Order of the Flow

These commands are for **RKE2**. On VKS and EKS, use the same `kubectl` checks; the node-level checks only apply where you have node access.

```bash
export PATH=$PATH:/var/lib/rancher/rke2/bin KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml

# Step 1 — DNS
kubectl run t --rm -it --image=busybox -- nslookup backend

# Step 2 — kernel prerequisites
lsmod | grep -E 'br_netfilter|nf_conntrack'
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward

# Step 3 — kube-proxy and its rules
kubectl -n kube-system get pods -o wide | grep kube-proxy   # static pods (RKE2)
kubectl -n kube-system get ds kube-proxy                    # DaemonSet (VKS/EKS)
curl -s 127.0.0.1:10256/healthz
kubectl get endpointslices -l kubernetes.io/service-name=backend
iptables-save | grep 10.43.0.50

# Step 4 — conntrack
conntrack -E -d 10.43.0.50          # watch live while you curl
conntrack -C; sysctl net.netfilter.nf_conntrack_max

# Step 5 — CNI
kubectl -n kube-system get pods -o wide | grep canal
ip route | grep -E 'cali|flannel.1'
```

**Changing kube-proxy flags in RKE2:** edit `kube-proxy-arg` in `/etc/rancher/rke2/config.yaml`, then run `systemctl restart rke2-server` (or `rke2-agent`). Don't edit the manifest directly; RKE2 overwrites it on restart.

---

## Quick Reference (RKE2 defaults)

| Item | Value |
|---|---|
| Pod CIDR | `10.42.0.0/16` |
| Service CIDR | `10.43.0.0/16` |
| `kubernetes` Service | `10.43.0.1` |
| CoreDNS Service | `10.43.0.10` |
| NodePort range | `30000–32767` |
| Canal VXLAN | UDP `8472`, interface `flannel.1` |
| kube-proxy health | `127.0.0.1:10256/healthz` |
| kube-proxy mode | `127.0.0.1:10249/proxyMode` |
