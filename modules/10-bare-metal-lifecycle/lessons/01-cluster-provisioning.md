---
lesson: "10.1"
title: "Cluster provisioning: bootstrap a control plane from nothing"
module: "10"
concept: "Cluster provisioning: bootstrap a control plane from nothing"
status: not-started
est_time: "10h"
prev: null
next: "02-etcd-operations.md"
artifacts: []
sources: 9
---

# 10.1 · Cluster provisioning: bootstrap a control plane from nothing

> **Concept.** Generate the PKI by hand, wire the control-plane static pods, join a worker — and name every certificate and which component presents it to which peer.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

This is lesson one of the module and the first lesson of the whole course to put you on bare
metal with no managed control plane underneath you. Everything before this assumed a cluster
already existed — you consumed one, tuned workloads on one, reasoned about its controllers. This
lesson removes that floor: you generate the trust roots, mint the certs, and stand up the
apiserver/etcd/controller-manager/scheduler yourself, so that from here on "the control plane" is
a set of files and processes you built, not a button you clicked. Everything downstream in this
module — etcd ownership (10.2), HA topology (10.3), declarative fleets (10.4), node provisioning
(10.5) — assumes you can already answer "what cert does X present to Y and why," because that
question is the one that pages you at 2am when any of those later systems misbehaves.

## Why this matters

This is the actual job at a bare-metal GPU shop: no EKS "Create cluster" button provisions a
control plane you never see. NVIDIA's Sr SSE (Kubernetes Node Lifecycle, DGX Cloud) posting asks
candidates to "build and refine CAPI providers... for scalable node provisioning" against bare
metal; CoreWeave's Kubernetes Platforms team explicitly owns "provisioning bare-metal and virtual
clusters with Cluster API" and runs its own CKS "directly on bare metal, without a hypervisor."
Both assume you already know how a control plane comes into existence by hand — CAPI providers
and kubeadm are automation *over* the process this lesson makes you do manually.

The cost of not knowing it is concrete and it repeats in the wild: a self-managed cluster whose
apiserver serving certificate quietly expired at the one-year kubeadm default froze every
deployment and killed `kubectl` cluster-wide, with no warning until it happened (see
[Real-world use cases](#real-world-use-cases) below) — an independent postmortem from a different
team hit the identical failure class the same year. When the apiserver won't come up because a SAN
is missing, or a kubelet is `Unauthorized`, you debug it by knowing exactly which cert is
presented in which direction — nobody hands you that diagram; you have to have built one.

## What's new here (calibration)

Per this module's calibration (see the [README](../README.md#calibrated-to-your-background---what-we-skip)):
you already have **02**'s control-plane anatomy (what etcd/apiserver/controller-manager/scheduler
*are* and how they relate at runtime), **04**'s GPU Operator and driver-rollout experience, **05**'s
XID/NPD concepts, and general on-prem/PXE/colocation fluency — none of that is re-taught here.

What's genuinely new in this lesson:
- The **PKI graph** — every CA, leaf cert, its Subject/CN, its SANs, its `extendedKeyUsage`
  (server vs client auth), and its issuer. None of this is in 02, which treated etcd/apiserver as
  black boxes that "just talk to each other."
- **kubeconfig files as credential bundles**, not just connection strings.
- **Static pods** as the bootstrap mechanism that solves the control-plane chicken-and-egg problem.
- **Bootstrap tokens + TLS bootstrapping** — how a worker gets its first cert without hand-copied
  PEMs.
- The **kubeadm diff** — running it by hand first, then `kubeadm init`, to see precisely what gets
  automated, which is the artifact a Staff-level interview probe ("bootstrap a control plane by
  hand — name every cert") is actually testing for.

## Core concepts

### The PKI, end to end
Kubernetes mTLS rests on (typically) **three independent CAs**. They are separate trust roots on purpose — compromising the front-proxy CA must not let you mint apiserver-trusted client certs.

| CA | Signs | Location (kubeadm default) |
|----|-------|-----------------------------|
| `kubernetes` (cluster CA) | apiserver serving cert, apiserver→etcd client cert (if sharing), apiserver→kubelet client cert, admin/controller-manager/scheduler/kubelet client certs | `/etc/kubernetes/pki/ca.crt`,`ca.key` |
| `etcd/ca` | etcd server certs, etcd peer certs, the apiserver-etcd-client cert | `/etc/kubernetes/pki/etcd/ca.crt` |
| `front-proxy-ca` | `front-proxy-client` (aggregation layer / extension apiservers) | `/etc/kubernetes/pki/front-proxy-ca.crt` |

Plus a keypair that is **not a cert at all**: the **ServiceAccount signing key** (`sa.key`/`sa.pub`). The controller-manager (and, for projected/bound tokens, the apiserver's `--service-account-signing-key-file`) signs SA JWTs with `sa.key`; the apiserver verifies them with `sa.pub`. No CA, no expiry — just an asymmetric keypair. If `sa.pub` and `sa.key` ever drift apart across control-plane nodes, every workload's token silently fails verification on one apiserver and works on another — a maddening intermittent-auth bug unique to hand-built/HA setups.

The leaf certs and, critically, **who presents each to whom**:

- **`apiserver` serving cert** (`apiserver.crt`) — presented BY the apiserver TO every client (kubectl, kubelets, controllers). `extendedKeyUsage: serverAuth`. Its **SANs are the failure point**: must list the API service ClusterIP (e.g. `10.96.0.1`), `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.default.svc.cluster.local`, the node hostname, and every IP/DNS name clients reach it by (including the load balancer VIP for HA). Missing SAN ⇒ `x509: certificate is valid for X, not Y`.
- **`apiserver-etcd-client`** (`apiserver-etcd-client.crt`) — presented BY the apiserver TO etcd, as a **client** cert (`clientAuth`), signed by the **etcd CA** (etcd only trusts its own CA). This is the answer to self-check (a), direction one.
- **`apiserver-kubelet-client`** (`apiserver-kubelet-client.crt`) — presented BY the apiserver TO each kubelet, as a client cert, when the apiserver initiates `kubectl logs`/`exec`/`port-forward` or metrics scrapes. Its CN (`kube-apiserver-kubelet-client`, group `system:masters` in kubeadm) is what the kubelet authorizes. This is self-check (a), direction two.
- **`front-proxy-client`** — presented BY the apiserver (acting as aggregator) TO extension apiservers (metrics-server, custom APIs). Verified via `requestheader-client-ca-file`.
- **etcd `server.crt` / `peer.crt`** — etcd presents `server` to the apiserver, `peer` to other etcd members (peer certs are both server- and client-auth because peer connections are bidirectional).
- **kubelet client cert** (`/var/lib/kubelet/pki/kubelet-client-current.pem`) — presented BY each kubelet TO the apiserver. CN `system:node:<nodename>`, group `system:nodes` — the Node authorizer keys off exactly this.
- **kubelet serving cert** (`kubelet.crt`) — presented BY the kubelet TO the apiserver when the apiserver connects inbound (logs/exec).

Note the two independent directions on the apiserver↔kubelet link: apiserver→kubelet uses `apiserver-kubelet-client`; kubelet→apiserver uses the kubelet's own `system:node:` client cert. Different certs, opposite directions.

### Generating the certs — the actual commands
In KTHW you drive `cfssl`; the mental model is the same with `openssl`. A leaf cert is: a keypair, a CSR carrying the CN/O and SANs, signed by a CA with a profile that pins the `extendedKeyUsage`. The apiserver serving cert, for example:
```json
// csr: CN plus every name a client might dial
{ "CN": "kube-apiserver",
  "hosts": ["10.96.0.1","127.0.0.1","<node-ip>","<lb-vip>",
            "kubernetes","kubernetes.default","kubernetes.default.svc",
            "kubernetes.default.svc.cluster.local"] }
```
```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=kubernetes \
  apiserver-csr.json | cfssljson -bare apiserver     # -> apiserver.pem / apiserver-key.pem
```
The `kubernetes` profile sets `server auth`; a `client` profile sets `client auth`. Get the EKU wrong and a client cert is rejected as "not valid for client authentication" even though the CA is right. Inspect any result with `openssl x509 -in apiserver.pem -noout -text` and read the SAN and EKU blocks — this is the single most common bootstrap bug.

### kubeconfigs are credential envelopes
A kubeconfig is not just a URL — it embeds `client-certificate-data` + `client-key-data` (base64 PEM) plus the `certificate-authority-data` to trust the apiserver. kubeadm generates four under `/etc/kubernetes/`:
- `admin.conf` — client cert CN `kubernetes-admin`, group `system:masters` (cluster-admin via the default binding).
- `kubelet.conf` — the node's kubelet client identity, CN `system:node:<name>`.
- `controller-manager.conf` — CN `system:kube-controller-manager`.
- `scheduler.conf` — CN `system:kube-scheduler`.
RBAC for the built-in components is bound to these exact CNs via `system:kube-controller-manager` / `system:kube-scheduler` ClusterRoles. Build a kubeconfig by hand with `kubectl config set-cluster/set-credentials/set-context --embed-certs=true` — doing it once demystifies what `admin.conf` actually is.

### Control-plane components run as static pods — and why
The kubelet watches `--pod-manifest-path` (default `/etc/kubernetes/manifests/`) and runs any pod manifest it finds there **directly, without an apiserver**. kubeadm drops `kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`, and (stacked) `etcd.yaml` there.

Why static pods and not Deployments (self-check c): a Deployment is reconciled by the controller-manager, which talks to the apiserver, which needs etcd — none of which exist yet at bootstrap. Static pods break the chicken-and-egg: the kubelet is the one component that can run a container with only a file on disk. The apiserver then creates read-only **mirror pods** so the static pods are *visible* via `kubectl get pods -n kube-system`, but you cannot manage them through the API — edit the manifest file and the kubelet re-applies. (In pure KTHW you instead run the components as **systemd units**, which is the same idea one level lower: no orchestrator, just the init system.)

### The Node authorizer and admission — why the kubelet's CN is load-bearing
Once a kubelet presents its `system:node:<name>` client cert (group `system:nodes`), two mechanisms constrain it: the **Node authorizer** (`--authorization-mode=Node,RBAC`) lets a node read/write only objects tied to *its own* pods (its Secrets, ConfigMaps, its Node status), and the **NodeRestriction admission plugin** stops a compromised kubelet from editing other nodes or labeling itself into privileged scheduling. This is why the CN must be exactly `system:node:<nodename>` and the group exactly `system:nodes` — the authorizer parses the node name straight out of the cert. A cert with the wrong CN authenticates fine but is authorized for nothing; a cert with `system:masters` would (dangerously) be cluster-admin. On managed clusters this was invisible; here you set the apiserver flags and sign the certs that make it work.

### HA: fronting multiple apiservers
With more than one control-plane node, clients and kubelets hit a **load balancer VIP**, not one apiserver. Consequences you own: the VIP/DNS name **must** be in every apiserver serving cert's SANs (add it before init, or certs won't validate through the LB), and the LB does **L4 TCP passthrough** — it must not terminate TLS, because mTLS is end-to-end to the apiserver. `kubeadm init --control-plane-endpoint=<vip>:6443` bakes this in; forgetting it is why single-node kubeadm clusters are painful to convert to HA later. Lesson 10.3 builds this out fully with kube-vip.

### Bootstrap tokens and TLS bootstrapping
Hand-copying a signed kubelet cert to every one of hundreds of GPU nodes does not scale. Instead:
- A **bootstrap token** is a short-lived secret of the form `[a-z0-9]{6}.[a-z0-9]{16}` — a public **Token ID** and a secret. It lives as a Secret `bootstrap-token-<id>` in `kube-system`, with an expiry (default 24h) and `usage-bootstrap-*` flags (self-check b: it authenticates a joining node just long enough to submit a CSR). The token maps to the group `system:bootstrappers:...`.
- The joining kubelet uses the token to authenticate to the apiserver and submit a **CertificateSigningRequest** for its own `system:node:<name>` client cert. RBAC lets bootstrappers auto-create CSRs; the CSR-approver controller (or you) approves it; the kubelet downloads the signed cert and rotates it thereafter. `kubeadm join` wires this whole flow.

### Topology: stacked vs external etcd
Two HA shapes, and the choice changes your PKI and your blast radius:
- **Stacked** — etcd runs as a static pod *on each control-plane node*, colocated with the apiserver. Fewer machines; but losing a control-plane node loses an etcd member too, so a 3-node stacked HA cluster tolerates only one node failure.
- **External** — etcd is its own dedicated cluster (its own CA, its own machines, its own fast disks). The apiserver reaches it purely as a client via `apiserver-etcd-client`. More hardware, but etcd failures and control-plane failures are decoupled — the topology big GPU fleets prefer, and the one where lesson 10.2's disk/quorum ownership is sharpest.

Either way the apiserver→etcd link is a **client cert signed by the etcd CA**; that fact is topology-independent.

### The CRI and why the apiserver never talks to it
The kubelet talks to the container runtime (**containerd**/CRI-O) over the **CRI** gRPC socket (`/run/containerd/containerd.sock`) — no TLS, a local Unix socket. The apiserver never touches the runtime; it tells the kubelet the desired state and the kubelet drives containerd. On bare metal you install and configure containerd yourself (cgroup driver must be `systemd` to match the kubelet, or pods flap). This is the layer EKS/GKE pre-baked into the node AMI.

### Certificate lifetime — and why it fails slowly, not all at once
kubeadm leaf certs default to **1 year**; the CAs to 10 years. Expired apiserver/kubelet certs are a classic silent 2am outage on clusters nobody upgraded — the apiserver refuses connections and nothing self-heals because renewal itself needs a working API. Two independent public postmortems converge on this exact failure class: one describes a self-managed cluster whose apiserver certificate expired at the 1-year default, freezing all deployments and killing `kubectl` cluster-wide with no warning; another, an independent December-2019 write-up, walks the same failure with its own diagnosis steps (start from `x509` errors, work backwards to `kubeadm certs check-expiration`). See [Real-world use cases](#real-world-use-cases).

The sharper trap for HA clusters built incrementally: because each cert is minted the day it's generated, and control-plane nodes are often added on *different days* (a node re-joined after maintenance, a new CP node added months later), their certs expire on *different days too* — producing a slow drip of failures (one etcd peer cert first, then another apiserver, weeks apart) rather than one clean simultaneous outage. That drip is *harder* to diagnose than a single hard failure because the cluster looks "mostly fine" while it's actually degrading.

`kubeadm certs check-expiration` reports every cert's expiry; `kubeadm certs renew all` rotates the leaves (restart the static pods after). Kubelet **client** certs auto-rotate via CSR when TLS-bootstrapped; **serving** certs need `RotateKubeletServerCertificate` plus CSR approval. Put `check-expiration` in a monthly cron on every control-plane node — this is the cheapest outage you will ever prevent.

At telco/edge scale, PKI ownership is a first-class platform concern, not an afterthought: Deutsche Telekom's ["Das Schiff"](https://github.com/telekom/das-schiff) CaaS platform runs Cluster API + Metal3 (+ vSphere) + Flux across hundreds of bare-metal edge locations in Germany, engineered so clusters keep running on correctly-issued certs and reconciled GitOps state even when the management plane is unreachable. That's the same PKI-ownership problem this lesson teaches, just at a scale where a manual `check-expiration` cron doesn't cut it — worth knowing this pattern exists even though the deep dive on CAPI/Metal3 itself is lesson 10.4's territory.

### Encryption at rest — a bare-metal concern EKS hid
On managed control planes the provider encrypted etcd's disk for you (often with a KMS key). On bare metal, **Secrets sit in etcd in plaintext by default** — anyone with the etcd data dir or a snapshot can read every credential. You wire this yourself with an **`EncryptionConfiguration`** passed to the apiserver via `--encryption-provider-config`:
```yaml
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc: { keys: [{ name: k1, secret: <base64-32B> }] }
      - identity: {}          # fallback: read plaintext during migration
```
Provider **order matters**: the first entry encrypts writes; `identity` last lets you read pre-existing plaintext while migrating (`kubectl get secrets -A -o json | kubectl replace -f -` rewrites them encrypted). For production, prefer a **KMS provider** so the key isn't in a file next to etcd. This is a direct line item in the deliverable's on-prem-vs-cloud economics: you now own key management the cloud used to.

### Mapping `kubeadm init` to phases — this IS your diff
`kubeadm init` is a pipeline of **phases** you can run individually (`kubeadm init phase --help`), and each phase maps to a chunk of your hand-built work:
| Phase | What it does (your Pass-1 equivalent) |
|-------|----------------------------------------|
| `certs` | generates the whole PKI you built with cfssl |
| `kubeconfig` | writes admin/kubelet/controller-manager/scheduler.conf |
| `control-plane` | drops the static-pod manifests (you wrote systemd units) |
| `etcd` | stacked-etcd static pod |
| `bootstrap-token` | creates the join token + RBAC for bootstrappers |
| `upload-config` | stores cluster config as a ConfigMap |
| `mark-control-plane` | taints/labels the node |
Running `kubeadm init phase certs all --dry-run` prints exactly what it would generate — the fastest way to produce the "what kubeadm automated" table.

### Bootstrap error → cert at fault (field cheat-sheet)
Almost every bootstrap failure is one cert or one SAN. Memorize the mapping:
| Error you see | Cert / cause |
|---------------|--------------|
| `x509: certificate is valid for X, not Y` | apiserver serving cert **missing a SAN** (the VIP or a node address) |
| `x509: certificate signed by unknown authority` (apiserver→etcd) | `apiserver-etcd-client` signed by the **wrong CA** (must be the etcd CA) |
| etcd `remote error: tls: bad certificate` | apiserver's etcd client cert/key mismatch or wrong CA |
| kubelet `Unauthorized` to apiserver | kubelet client cert CN not `system:node:<name>` / not in `system:nodes` |
| apiserver→kubelet `x509` on `kubectl logs` | `apiserver-kubelet-client` missing or kubelet's `--client-ca-file` wrong |
| SA tokens rejected | `sa.pub` on the apiserver doesn't match the `sa.key` the controller-manager signs with |
| aggregated API `401` | `front-proxy-client` / `requestheader-*` misconfig |
Reach for `openssl x509 -noout -text` on the named cert first; it's faster than reading apiserver logs blind.

## Perspectives

**The developer/consumer view.** You've spent a career on the other side of this: `kubectl` "just worked" because someone else's control plane presented a cert your client already trusted. That someone is now you. The payoff is that app-level auth bugs (a workload's ServiceAccount token being silently rejected, a webhook failing TLS verification) stop being mysterious once you can trace which cert, which CA, and which SAN is in play.

**The operator view.** Your job is no longer "does the cluster work" but "will the cluster still work in 11 months." Cert lifetime, SAN completeness for every future LB/VIP, and a monthly `check-expiration` cron are now yours. This is boring, unglamorous work — and it is exactly the work that silently protects uptime, which is why it shows up in interview probes.

**The security/PKI view.** Three independent CAs are a deliberate blast-radius decision: compromising the front-proxy CA (which only signs one client cert) must not let an attacker mint apiserver-trusted identities. The `sa.key`/`sa.pub` pair sitting outside any CA is a reminder that not all trust in the cluster is cert-based — some of it is a bare keypair you must protect and keep synchronized by hand across every control-plane node.

**The economics view.** Every one of these PKI and encryption-at-rest responsibilities was previously bundled into your managed-control-plane bill. When you build the capex-vs-cloud model in lesson 10.8, "self-managed control plane" needs a line item for the *engineering time* to run this cron, rotate these certs, and own this key management — not just hardware capex. A cluster that silently outages from an expired cert is not cheaper than EKS; it's a deferred cost that lands as an incident.

## Real-world use cases

- **"When Kubernetes Certificates Expire: A Production War Story"** — <https://medium.com/@olanipekunadekunleoluwole/when-kubernetes-certificates-expire-a-production-war-story-3bd4a54db3bf> — a self-managed cluster's apiserver certificate hit the 1-year kubeadm default, expired, and froze every deployment while `kubectl` died cluster-wide. Shows the exact "nobody rotated the leaf cert" failure this lesson's cert-lifetime section warns about, as it actually happened in production.
- **"2019-12 K8s certificate expiration outage"** — <https://vadosware.io/post/2019-12-k8s-cert-expiration-outage/> — an independent postmortem of the same failure class, with concrete diagnosis steps. Pairs directly with this lesson's "bootstrap error → cert at fault" cheat sheet, generalized from bootstrap-time to steady-state ops: the same table you use to debug a fresh `kubeadm init` also triages a cluster that's been running fine for a year.
- **Deutsche Telekom "Das Schiff"** — <https://github.com/telekom/das-schiff> — DT's CaaS platform runs CAPI + Metal3 (+ vSphere) + Flux across hundreds of bare-metal locations in Germany, engineered so clusters keep serving traffic on correctly-issued certs and reconciled GitOps state even when the management plane is unreachable. Shows PKI/bootstrap ownership at telco/edge fleet scale — a brief cross-reference here; the full CAPI/Metal3 story is lesson 10.4's.

## Worked example — tracing one apiserver→etcd write
A `kubectl apply -f pod.yaml` on a hand-built cluster:
1. kubectl loads `admin.conf`, presents client cert CN `kubernetes-admin` over TLS to `https://<apiserver>:6443`. The apiserver presents `apiserver.crt`; kubectl validates it against `ca.crt` and checks the SAN covers the address used.
2. Apiserver authenticates the client cert against `--client-ca-file=ca.crt`, extracts CN/O → user `kubernetes-admin`, group `system:masters`. Authorization: `system:masters` is hard-wired to allow-all.
3. Apiserver validates, defaults, runs admission, then must persist. It opens a client TLS connection to etcd on `:2379`, presenting **`apiserver-etcd-client.crt`** (signed by the etcd CA). etcd presents `server.crt`; the apiserver validates it against `/etc/kubernetes/pki/etcd/ca.crt`.
4. etcd authenticates the apiserver's client cert against its own CA, writes the key under `/registry/pods/...`, replicates to peers over `peer.crt`, and acks after quorum.
5. Watchers (informers in controller-manager, scheduler) receive the event over their own component kubeconfigs. Scheduler binds; kubelet on the target node — authenticated by its `system:node:` cert — pulls and runs.

Every hop above is a distinct cert in a distinct direction. If step 3 fails with `remote error: tls: bad certificate`, you signed `apiserver-etcd-client` with the wrong CA.

## Practice
Two passes on **3 cheap VMs** (KVM/multipass, or three `e2-small`/`t3.small`); one machine with multipass VMs is fine.

**Pass 1 — Kubernetes The Hard Way (hand-built).** Follow KTHW end to end:
- Generate the CA and every cert with `cfssl`/`openssl`: cluster CA, apiserver serving cert (get the SAN list right — ClusterIP + all node/LB addresses + the `kubernetes.default.svc...` names), `apiserver-kubelet-client`, kubelet certs, the four kubeconfigs, the etcd CA + server/peer certs + `apiserver-etcd-client`, front-proxy CA + client, and the `sa` keypair.
- Bring up etcd, then the apiserver/controller-manager/scheduler as **systemd units**, then join workers.
- For each cert, write down: filename, CN, group/O, SANs, EKU (serverAuth/clientAuth), signing CA, and **who presents it to whom**. This table is your deliverable artifact.

**Pass 2 — kubeadm, then diff.** On fresh VMs: install containerd (systemd cgroup driver), then `kubeadm init --control-plane-endpoint=<addr>:6443 --pod-network-cidr=10.244.0.0/16`, install a CNI, and `kubeadm join` a worker with the printed bootstrap token. Then diff against Pass 1:
- `ls -R /etc/kubernetes/pki` and `kubeadm certs check-expiration` — map every file back to your Pass-1 table and note the 1-year leaf expiries.
- `ls /etc/kubernetes/manifests` — note the control plane is **static pods** here vs systemd units in Pass 1; `crictl ps` shows them running under containerd.
- `kubectl -n kube-system get secrets | grep bootstrap-token`, inspect one, and watch `kubectl get csr` during the join to see the TLS-bootstrap CSR get created and approved.
- `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text` — read the SAN list kubeadm computed and compare to the hosts you hand-listed in Pass 1.
- Run `kubeadm init phase certs all --dry-run` on a scratch host to dump the exact cert set — that output *is* the spine of your writeup table.

**Acceptance:** (1) a hand-built control plane that passes `kubectl get nodes` Ready with a joined worker; (2) a written **"what kubeadm automated" diff** — the cert table, the static-pod-vs-systemd difference, and the bootstrap-token/TLS-bootstrap flow kubeadm did for you. Both feed the deliverable's KTHW writeup, and both are inputs to the 10.8 capex model's "engineering time to run this yourself" line.

## Common pitfalls

- **Forgetting the LB VIP in the apiserver SAN list before `kubeadm init`.** Certs are only regenerated with `kubeadm certs renew` on their normal fields, not automatically expanded with new SANs — retrofitting HA onto a single-node cluster after the fact usually means regenerating `apiserver.crt` by hand or reinitializing.
- **Signing `apiserver-etcd-client` with the cluster CA instead of the etcd CA.** It's the single most common bootstrap error in this lesson's cheat sheet (`x509: certificate signed by unknown authority`) — etcd only trusts its own CA, never the cluster CA, even though both are "Kubernetes" CAs conceptually.
- **Assuming 1-year cert expiry is "someone else's problem" like it was on EKS.** The two independent postmortems above show this is the single most common silent bare-metal outage. Put `kubeadm certs check-expiration` on a calendar, not in your memory.
- **Treating an incrementally-built HA cluster's cert expiries as synchronized.** Because control-plane nodes are often added weeks or months apart, their leaf certs expire on different days — expect a slow drip of single-member failures, not one clean cluster-wide outage, and check *every* node's expiry, not just the first one you built.
- **Confusing kubelet client-cert auto-rotation with serving-cert rotation.** Client certs (used to talk *to* the apiserver) rotate automatically via CSR once TLS bootstrapping is set up; serving certs (presented *to* the apiserver on inbound `logs`/`exec`) need `RotateKubeletServerCertificate` enabled and CSR approval — assuming both auto-rotate the same way leaves the serving cert to expire unnoticed.

## Self-check
**(a) Which certificate does the apiserver present to etcd, and which to the kubelet?**
**Answer:** To etcd it presents **`apiserver-etcd-client`** (client-auth cert signed by the **etcd CA**). To the kubelet it presents **`apiserver-kubelet-client`** (client-auth cert signed by the cluster CA) when the apiserver initiates logs/exec/metrics. Opposite direction on the same link, the kubelet presents its own `system:node:<name>` client cert to the apiserver.

**(b) What's inside a bootstrap token and what is it for?**
**Answer:** A `<6-char id>.<16-char secret>` string stored as Secret `bootstrap-token-<id>` in `kube-system`, with an expiry and `usage-bootstrap-authentication/signing` flags, mapping to group `system:bootstrappers`. It authenticates a **joining node** just long enough to submit a kubelet-client CSR under TLS bootstrapping — so you never hand-copy per-node certs.

**(c) Why are the control-plane components static pods rather than Deployments?**
**Answer:** A Deployment is reconciled by the controller-manager → apiserver → etcd, none of which exist at bootstrap (chicken-and-egg). Static pods are run by the **kubelet directly from a manifest file** with no apiserver, so they can bring the API up. The apiserver later shows read-only mirror pods for visibility.

**(d) Why do incrementally-built HA clusters tend to fail with a "slow drip" of cert-expiry incidents instead of one clean outage?**
**Answer:** Each leaf cert's 1-year clock starts the day it's minted. Control-plane nodes added at different times (initial bootstrap, a later-added third CP node, a re-join after maintenance) get certs minted on different days, so they expire on different days too — one etcd peer cert first, an apiserver cert weeks later, and so on — which is harder to diagnose than a single simultaneous failure because the cluster looks mostly healthy in between.

## Connections & what's next

This lesson gave you the trust roots and the bootstrap mechanism; it deliberately stopped at "one cert per component, one node at a time." Three threads pick up from here: **10.2 (etcd operations)** takes the etcd you just stood up and makes you own its disk, its quorum, and its 2am restore — the PKI here is a prerequisite (you need `apiserver-etcd-client` and the etcd CA working before you can even reach etcd to break it on purpose). **10.3 (control-plane HA)** takes the single-node PKI you built here and grows it to 3 nodes behind a VIP, which is exactly why the SAN list matters now. **10.4 (declarative fleets: CAPI + Talos)** is where the manual cert-minting in this lesson gets automated away by a provider — you'll appreciate what CAPI is doing for you precisely because you did it by hand first.

Next: **[10.2 · etcd operations](02-etcd-operations.md)** — you now know how the apiserver reaches etcd; the next lesson is about what happens to that etcd once it's yours to run.

## References & further reading

**Primary sources**
- **Kubernetes The Hard Way** — <https://github.com/kelseyhightower/kubernetes-the-hard-way> — the canonical from-scratch bootstrap; every cert and unit by hand. Read for: doing Pass 1 step by step.
- **kubeadm PKI certificates reference** — <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubeadm-certs/> — the authoritative table of every cert, its CA, CN/SAN, and renewal. Read for: the cert graph this lesson's tables summarize.
- **Certificate Management with kubeadm** — <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/> — `check-expiration`, `renew`, and automatic vs manual rotation. Read for: the monthly-cron habit this lesson pushes.
- **TLS bootstrapping** — <https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/> — the CSR/bootstrap-token flow in detail. Read for: grounding self-check (b).

**Real-world engineering blogs**
- **"When Kubernetes Certificates Expire: A Production War Story"** — <https://medium.com/@olanipekunadekunleoluwole/when-kubernetes-certificates-expire-a-production-war-story-3bd4a54db3bf> — what it shows: a real cluster-wide freeze from one expired apiserver cert.
- **"2019-12 K8s certificate expiration outage"** — <https://vadosware.io/post/2019-12-k8s-cert-expiration-outage/> — what it shows: an independent postmortem of the same failure class, with a diagnosis walkthrough.
- **Deutsche Telekom "Das Schiff"** — <https://github.com/telekom/das-schiff> — what it shows: PKI/bootstrap ownership generalized to hundreds of bare-metal edge sites.

**Deeper dives**
- **kubeadm implementation details** — <https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/> — the under-the-hood documentation of exactly what `kubeadm init`'s phases do; the natural next read after your Pass-2 diff.
- **cfssl** — <https://github.com/cloudflare/cfssl> — the CA/cert toolkit KTHW uses; worth reading its README once to understand CSR profiles and the `-config`/`-profile` mechanism beyond copy-pasting commands.
