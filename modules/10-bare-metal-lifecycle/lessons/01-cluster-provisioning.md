---
lesson: "10.1"
title: "Cluster provisioning: bootstrap a control plane from nothing"
module: "10"
concept: "Cluster provisioning: bootstrap a control plane from nothing"
status: not-started
est_time: "8h"
artifacts: []
---

# 10.1 · Cluster provisioning: bootstrap a control plane from nothing

> **Concept.** Generate the PKI by hand, wire the control-plane static pods, join a worker — and name every certificate and which component presents it to which peer.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters
This is the actual job at a bare-metal GPU shop: no EKS "Create cluster" button provisions a control plane you never see. When the apiserver won't come up because a SAN is missing from its serving cert, or a kubelet is `Unauthorized` against the apiserver, you debug it by knowing exactly which cert is presented in which direction. You have consumed 40+ managed control planes; here you build one from a directory of `.pem` files and a handful of systemd units and static-pod manifests, so the PKI stops being a black box.

## What's new here
Module 02 taught the **anatomy**: what etcd is, that the apiserver is the only client that talks to etcd, what informers and controllers do, how the scheduler binds pods. It answered *what the pieces are and how they relate at runtime*.

This lesson is **provisioning**: how those pieces come to exist on cold hardware. New material:
- The **PKI graph** — every CA, leaf cert, its Subject/CN, its SANs, its `extendedKeyUsage` (server vs client auth), and its issuer. This is not in 02 at all.
- **kubeconfig files as credential bundles** — admin, kubelet, controller-manager, scheduler each get their own client cert embedded in a kubeconfig.
- **Static pods** as the bootstrap mechanism for the control plane, run by the kubelet directly with no apiserver involved (chicken-and-egg).
- **Bootstrap tokens + TLS bootstrapping** — how a brand-new worker gets its first kubelet client cert without you hand-copying one per node.
- The **kubeadm diff**: after doing it by hand, run `kubeadm init` and see precisely which of these steps it automated.

## Core notes

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
With more than one control-plane node, clients and kubelets hit a **load balancer VIP**, not one apiserver. Consequences you own: the VIP/DNS name **must** be in every apiserver serving cert's SANs (add it before init, or certs won't validate through the LB), and the LB does **L4 TCP passthrough** — it must not terminate TLS, because mTLS is end-to-end to the apiserver. `kubeadm init --control-plane-endpoint=<vip>:6443` bakes this in; forgetting it is why single-node kubeadm clusters are painful to convert to HA later.

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

### Certificate lifetime
kubeadm leaf certs default to **1 year**; the CAs to 10 years. Expired apiserver/kubelet certs are a classic silent 2am outage on clusters nobody upgraded — the apiserver refuses connections and nothing self-heals because renewal itself needs a working API. `kubeadm certs check-expiration` reports every cert's expiry; `kubeadm certs renew all` rotates the leaves (restart the static pods after). Kubelet **client** certs auto-rotate via CSR when TLS-bootstrapped; **serving** certs need `RotateKubeletServerCertificate` plus CSR approval. Put `check-expiration` in a monthly cron on every control-plane node — this is the cheapest outage you will ever prevent.

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

**Acceptance:** (1) a hand-built control plane that passes `kubectl get nodes` Ready with a joined worker; (2) a written **"what kubeadm automated" diff** — the cert table, the static-pod-vs-systemd difference, and the bootstrap-token/TLS-bootstrap flow kubeadm did for you. Both feed the deliverable's KTHW writeup.

## Self-check
**(a) Which certificate does the apiserver present to etcd, and which to the kubelet?**
**Answer:** To etcd it presents **`apiserver-etcd-client`** (client-auth cert signed by the **etcd CA**). To the kubelet it presents **`apiserver-kubelet-client`** (client-auth cert signed by the cluster CA) when the apiserver initiates logs/exec/metrics. Opposite direction on the same link, the kubelet presents its own `system:node:<name>` client cert to the apiserver.

**(b) What's inside a bootstrap token and what is it for?**
**Answer:** A `<6-char id>.<16-char secret>` string stored as Secret `bootstrap-token-<id>` in `kube-system`, with an expiry and `usage-bootstrap-authentication/signing` flags, mapping to group `system:bootstrappers`. It authenticates a **joining node** just long enough to submit a kubelet-client CSR under TLS bootstrapping — so you never hand-copy per-node certs.

**(c) Why are the control-plane components static pods rather than Deployments?**
**Answer:** A Deployment is reconciled by the controller-manager → apiserver → etcd, none of which exist at bootstrap (chicken-and-egg). Static pods are run by the **kubelet directly from a manifest file** with no apiserver, so they can bring the API up. The apiserver later shows read-only mirror pods for visibility.

## Resources
1. **Kubernetes The Hard Way** — https://github.com/kelseyhightower/kubernetes-the-hard-way — the canonical from-scratch bootstrap; every cert and unit by hand. **Deep** (this is Pass 1). Why: nothing else forces you to name every PEM.
2. **kubeadm PKI certificates reference** — https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubeadm-certs/ — the authoritative table of every cert, its CA, CN/SAN, and renewal. **Deep** for the cert graph. Why: your Pass-2 diff is basically this page.
3. **TLS bootstrapping** — https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/ — the CSR/bootstrap-token flow in detail. **Skim.** Why: grounds self-check (b).
