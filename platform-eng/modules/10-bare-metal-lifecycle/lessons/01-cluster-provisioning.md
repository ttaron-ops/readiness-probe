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
sources: 10
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

- **X.509 as an authentication mechanism**, not a checkbox — how `CN`/`O` become a Kubernetes
  user and groups, what `extendedKeyUsage` actually gates, and the exact rule a client uses to
  match a hostname against a SAN.
- The **PKI graph** — every CA, leaf cert, its Subject/CN, its SANs, its `extendedKeyUsage`
  (server vs client auth), and its issuer, reproduced from the kubeadm source. None of this is in
  02, which treated etcd/apiserver as black boxes that "just talk to each other."
- **kubeconfig files as credential bundles**, not just connection strings — including the
  `admin.conf` / `super-admin.conf` split that landed in kubeadm v1.29 and that most older
  material still gets wrong.
- **Static pods** as the bootstrap mechanism that solves the control-plane chicken-and-egg
  problem, down to the kubelet's file-watch loop and mirror-pod behaviour.
- **Bootstrap tokens + TLS bootstrapping** — the full CSR message sequence, the four RBAC objects
  kubeadm creates to make it work, and the two signers involved.
- The **kubeadm diff** — running it by hand first, then `kubeadm init`, to see precisely what gets
  automated, which is the artifact a Staff-level interview probe ("bootstrap a control plane by
  hand — name every cert") is actually testing for.
- **Provisioning arithmetic** — how long a fleet bring-up takes as a function of per-node time
  and parallelism, and where the serialisation points are.

**Version note.** Unless stated otherwise, all kubeadm constants, cert definitions, phase orders
and RBAC objects below were read directly from `cmd/kubeadm/` on the `release-1.36` branch of
`kubernetes/kubernetes` (the current stable line as of August 2026), not from documentation.
Where a behaviour changed at a specific release, the release is named. Command transcripts are
**representative** — formatted the way the real tool formats them, with realistic values — rather
than literal captures, except where explicitly attributed.

## Core concepts

### The bootstrap problem: what must exist before what

Before any certificate discussion, get the ordering constraint straight, because every design
decision below falls out of it.

A Kubernetes control plane is a set of processes that all authenticate to each other with TLS
client certificates, and all coordinate through one API server that stores everything in etcd.
That gives you a hard dependency chain:

- The apiserver cannot start until etcd is reachable *and* it holds a client certificate etcd
  trusts.
- etcd cannot start until it has a data directory, a server certificate, and (in a multi-member
  cluster) knows its peers.
- Nothing can hold a certificate until a CA exists to sign it.
- The controller-manager and scheduler cannot do anything until the apiserver is up.
- The kubelet is the *only* component that can run a container without an apiserver.

So the bring-up order is forced: **CA keys → leaf certs → kubeconfigs → etcd → apiserver →
controller-manager/scheduler → CNI → workers.** Every provisioning system you will ever meet —
KTHW's shell scripts, kubeadm's phases, Cluster API's `KubeadmControlPlane`, Talos's machine
config — is a different encoding of that same order. Once you have internalised the order, the
tools stop looking like magic and start looking like transcriptions.

Here is the whole pipeline from bare hardware, with the failure at each stage and how it presents:

```
 STAGE            WHAT HAPPENS                          WHAT FAILS HERE / HOW IT LOOKS
 ─────            ────────────                          ──────────────────────────────
 0 POWER    ┌──────────────────────────────┐   BMC unreachable; PSU/PDU fault.
   + BMC    │ Redfish/IPMI: power on,      │   Symptom: node never appears in DHCP logs at all.
            │ set boot device, open SOL    │   Debug: `ipmitool chassis power status`, BMC web UI.
            └──────────────┬───────────────┘
 1 FIRMWARE ┌──────────────▼───────────────┐   Wrong boot mode (legacy vs UEFI), Secure Boot on
            │ BIOS/UEFI POST; NIC option   │   with an unsigned kernel, PXE disabled on the NIC,
            │ ROM does PXE/HTTP boot       │   SR-IOV/IOMMU off (bites you later, not now).
            └──────────────┬───────────────┘   Symptom: "No boot device"; console is your only view.
 2 NETBOOT  ┌──────────────▼───────────────┐   DHCP offers no next-server; TFTP blocked;
            │ DHCP → next-server → iPXE →  │   wrong architecture binary (BIOS vs UEFI x64).
            │ kernel + initrd over HTTP    │   Symptom: PXE-E53/PXE-E32 timeouts on console.
            └──────────────┬───────────────┘
 3 IMAGE    ┌──────────────▼───────────────┐   Image checksum mismatch; disk selector matched the
            │ Installer writes OS image to │   wrong device (wiped the data disk); no space.
            │ disk, sets up partitions     │   Symptom: install aborts, or boots to the old OS.
            └──────────────┬───────────────┘
 4 RUNTIME  ┌──────────────▼───────────────┐   containerd cgroup driver != kubelet's; missing
            │ containerd/CRI-O installed,  │   CNI binaries; registry unreachable / no mirror.
            │ cgroup driver = systemd      │   Symptom: pods stuck ContainerCreating; kubelet
            └──────────────┬───────────────┘   flaps NotReady with `failed to get cgroup stats`.
 5 PKI      ┌──────────────▼───────────────┐   Wrong CA signed a leaf; missing SAN; wrong EKU.
            │ CAs generated, leaf certs    │   Symptom: `x509: certificate signed by unknown
            │ signed, kubeconfigs written  │   authority` / `certificate is valid for X, not Y`.
            └──────────────┬───────────────┘
 6 ETCD     ┌──────────────▼───────────────┐   Peer certs missing SANs for peer IPs; clock skew;
            │ etcd starts, forms cluster,  │   `--initial-cluster` disagrees between members.
            │ elects a leader              │   Symptom: no leader; "cluster ID mismatch".
            └──────────────┬───────────────┘
 7 CONTROL  ┌──────────────▼───────────────┐   apiserver can't reach etcd; bad flags; port in use.
   PLANE    │ kubelet runs static pods:    │   Symptom: kubelet restarts the pod in a loop;
            │ apiserver, CM, scheduler     │   `crictl logs` on the apiserver container tells you.
            └──────────────┬───────────────┘
 8 CNI      ┌──────────────▼───────────────┐   No CNI installed → every node stays NotReady.
            │ CNI DaemonSet lands, node    │   Symptom: `NetworkReady=false ... cni plugin not
            │ goes Ready                   │   initialized` in `kubectl describe node`.
            └──────────────┬───────────────┘
 9 JOIN     ┌──────────────▼───────────────┐   Expired bootstrap token; CA hash mismatch;
            │ Worker: token → CSR →        │   CSR never approved.
            │ signed kubelet client cert   │   Symptom: `Unauthorized`, or CSR stuck Pending.
            └──────────────────────────────┘
```

Stages 0–4 are lesson 10.5's territory in full detail. Stages 5–9 are this lesson. The reason to
show all ten is that on bare metal *you own all ten*, and the single most valuable diagnostic
skill is knowing which stage a symptom belongs to before you start reading logs.

### X.509, only the parts Kubernetes uses

You need four facts about certificates, and Kubernetes uses each of them in a specific way.

**1. A certificate binds a public key to a subject, signed by a CA.** The subject is an X.500
distinguished name. Kubernetes reads exactly two of its fields:

- **`CN` (Common Name) → the username.**
- **`O` (Organization), possibly repeated → the group memberships.**

That is the whole mapping. When kube-apiserver authenticates a client certificate that chains to
`--client-ca-file`, it constructs a `user.Info` with `Name = CN` and `Groups = [O...]` and hands
it to the authorizers. There is no lookup, no user database, no revocation check (Kubernetes does
not consult CRLs or OCSP). **Anyone who can get the cluster CA to sign a cert with
`O=system:masters` is cluster-admin, permanently, until you rotate the CA.** That single sentence
explains most of the PKI design decisions in this lesson.

**2. `extendedKeyUsage` (EKU) gates the *direction* the cert may be used in.** `serverAuth` means
"this cert may be presented by a TLS server"; `clientAuth` means "this cert may be presented by a
TLS client." Go's `crypto/tls` enforces this during verification, so a cert with only `serverAuth`
that you try to use as a client credential is rejected — with a message that names the usage, not
the CA, which is why "wrong EKU" is so often misdiagnosed as "wrong CA."

**3. SANs, not CN, decide hostname matching.** The `subjectAltName` extension holds DNS names and
IP addresses. A Go client dialling `https://10.10.0.100:6443` requires `10.10.0.100` to be present
as an **IP SAN** — an entry in `DNSNames` will not match an IP literal, and `CN` has been ignored
for hostname verification since Go 1.15. This is why the apiserver serving cert's SAN list is the
single most common bootstrap bug: it is fixed at generation time and it must anticipate every
address any client will ever dial.

**4. Validity is a wall-clock window, checked by both sides.** `notBefore`/`notAfter` are absolute
timestamps. Nothing renews them for you unless you build the renewal. A node whose clock is
skewed past `notAfter` fails exactly like an expired cert.

Two independent trust decisions happen on every mTLS connection, and conflating them is the root
of most confusion:

```
   CLIENT                                                   SERVER
   ──────                                                   ──────
   presents: client cert (EKU clientAuth)  ───────────────▶  verifies chain against
   e.g. apiserver-etcd-client.crt                            --client-ca-file / --trusted-ca-file
                                                             then: CN → user, O → groups

   verifies chain against  ◀───────────────  presents: server cert (EKU serverAuth)
   the CA bundle in its kubeconfig                           e.g. apiserver.crt
   ("certificate-authority-data")                            + SAN must cover the dialled address
   + SAN must cover the address it dialled
```

**The two directions use different certificates and different CA bundles.** That is why
"apiserver ↔ kubelet" is two certificates, not one, and why an apiserver that can serve `kubectl`
perfectly can still fail `kubectl logs`.

### The trust graph: three CAs and one keypair that is not a CA

kubeadm builds **three independent, self-signed CAs**. They are separate roots on purpose: each CA
is a blast radius. Compromising the front-proxy CA — which signs exactly one client certificate —
must not let an attacker mint an identity the apiserver trusts.

```
                        ┌──────────────────────────────────────────────┐
                        │  ca.crt / ca.key      CN=kubernetes  (10 yr) │
                        │  the CLUSTER CA                              │
                        └───┬──────────┬──────────┬──────────┬─────────┘
        signs (serverAuth)  │          │ clientAuth          │ clientAuth (via kubeconfigs)
                            ▼          ▼          ▼          ▼
                     apiserver.crt  apiserver-  kubelet   admin.conf / super-admin.conf
                     CN=kube-       kubelet-    client    controller-manager.conf
                     apiserver      client.crt  certs     scheduler.conf  kubelet.conf
                     SANs: see      CN=kube-    CN=system:node:<name>
                     below          apiserver-  O=system:nodes
                                    kubelet-
                                    client
                     ▲ presented BY apiserver TO every client
                                    ▲ presented BY apiserver TO each kubelet
                                               ▲ presented BY kubelet TO apiserver

                        ┌──────────────────────────────────────────────┐
                        │  etcd/ca.crt          CN=etcd-ca     (10 yr) │
                        └───┬──────────┬──────────┬───────────┬────────┘
                            ▼          ▼          ▼           ▼
                    etcd/server.crt  etcd/peer.crt  apiserver-   etcd/healthcheck-
                    CN=<nodename>    CN=<nodename>  etcd-client  client.crt
                    EKU: server+     EKU: server+   CN=kube-     CN=kube-etcd-
                         client           client    apiserver-   healthcheck-client
                                                    etcd-client  EKU: clientAuth
                    ▲ etcd→clients   ▲ etcd↔etcd    ▲ apiserver→etcd  ▲ liveness probe→etcd

                        ┌──────────────────────────────────────────────┐
                        │  front-proxy-ca.crt   CN=front-proxy-ca      │
                        └───────────────────┬──────────────────────────┘
                                            ▼
                                 front-proxy-client.crt
                                 CN=front-proxy-client, EKU: clientAuth
                                 ▲ presented BY apiserver (as aggregator)
                                   TO extension apiservers (metrics-server, …)

                        ┌──────────────────────────────────────────────┐
                        │  sa.key / sa.pub   —  NOT A CERTIFICATE      │
                        │  a bare RSA/ECDSA keypair, no CA, no expiry  │
                        └──────────────────────────────────────────────┘
                          controller-manager + apiserver SIGN ServiceAccount
                          JWTs with sa.key; apiserver VERIFIES with sa.pub
```

The `sa` keypair deserves the callout it gets. It is not a certificate, has no expiry, and is not
issued by anything. The apiserver signs bound ServiceAccount tokens with
`--service-account-signing-key-file` and verifies them with `--service-account-key-file`. **If
`sa.key` and `sa.pub` drift apart across control-plane nodes, every workload token verifies on
one apiserver and fails on another** — producing an intermittent 401 that follows the load
balancer's round-robin, not any pattern in your application. This failure mode is unique to
hand-built and incrementally-grown HA clusters; `kubeadm join --control-plane` copies the keypair
for you via the `kubeadm-certs` Secret, which is exactly why that mechanism exists.

Here is the complete leaf inventory, transcribed from `cmd/kubeadm/app/phases/certs/certlist.go`
and `cmd/kubeadm/app/constants/constants.go` (k8s 1.36):

| File under `/etc/kubernetes/pki/` | CN | O (groups) | EKU | Signed by | Presented **by** → **to** |
|---|---|---|---|---|---|
| `ca.crt` | `kubernetes` | — | CA | self | trust root for the cluster |
| `apiserver.crt` | `kube-apiserver` | — | serverAuth | cluster CA | apiserver → all clients |
| `apiserver-kubelet-client.crt` | `kube-apiserver-kubelet-client` | — | clientAuth | cluster CA | apiserver → kubelet |
| `front-proxy-ca.crt` | `front-proxy-ca` | — | CA | self | trust root for aggregation |
| `front-proxy-client.crt` | `front-proxy-client` | — | clientAuth | front-proxy CA | apiserver → extension apiservers |
| `etcd/ca.crt` | `etcd-ca` | — | CA | self | trust root for etcd |
| `etcd/server.crt` | `<node name>` | — | serverAuth **+ clientAuth** | etcd CA | etcd → its clients |
| `etcd/peer.crt` | `<node name>` | — | serverAuth + clientAuth | etcd CA | etcd ↔ etcd |
| `etcd/healthcheck-client.crt` | `kube-etcd-healthcheck-client` | — | clientAuth | etcd CA | static-pod liveness probe → etcd |
| `apiserver-etcd-client.crt` | `kube-apiserver-etcd-client` | — | clientAuth | etcd CA | apiserver → etcd |
| `sa.key` / `sa.pub` | n/a | n/a | n/a | n/a | SA token signing/verification |

Three details in that table are the ones people get wrong:

- **`apiserver-kubelet-client` carries no `O` at all** in current kubeadm. Older material claims
  `O=system:masters`; that is not what the source does. Its authorization comes from an explicit
  ClusterRoleBinding named `kubeadm:apiserver-kubelet-client`, binding the **user**
  `kube-apiserver-kubelet-client` to the built-in ClusterRole `system:kubelet-api-admin`. If you
  hand-build and forget that binding, `kubectl logs` returns 403 from the *kubelet*, not the
  apiserver.
- **`etcd/server.crt` has `clientAuth` as well as `serverAuth`.** That looks like a mistake and
  is not: etcd has required client-auth usage on its server certificate since 3.2
  (`etcd-io/etcd#9785`), and the kubeadm source carries a comment saying so. Strip `clientAuth`
  "to be tidy" and etcd fails to start.
- **The etcd server and peer certs use the node name as CN**, not a fixed string, because peer
  identity is per-machine.

The apiserver serving cert's SAN list is computed by `GetAPIServerAltNames()`, and it is worth
knowing exactly what goes in, because this is the list you must reproduce by hand:

| SAN | Where it comes from |
|---|---|
| `<node name>` (DNS) | `nodeRegistration.name` |
| `kubernetes`, `kubernetes.default`, `kubernetes.default.svc` (DNS) | hard-coded |
| `kubernetes.default.svc.<dnsDomain>` (DNS) | `networking.dnsDomain`, default `cluster.local` |
| first IP of the service CIDR, e.g. `10.96.0.1` (IP) | `networking.serviceSubnet` |
| the node's `advertiseAddress` (IP) | `localAPIEndpoint.advertiseAddress` |
| the `controlPlaneEndpoint` host — IP **or** DNS depending on what you wrote | `controlPlaneEndpoint` |
| everything in `apiServer.certSANs` | your config |

Note what is *not* automatically there: any additional VIP, any external DNS name, any NAT
address, any future load-balancer address. Those are your job, via `certSANs`, **before** the cert
is minted.

### Generating the PKI by hand

The mechanics are identical whichever tool you use: create a key, describe a subject and a SAN
set, sign with a CA under a profile that pins the EKU. With `openssl`, using a config file so the
SANs and EKU actually make it into the cert (the classic mistake is passing `-subj` and expecting
extensions to appear):

```bash
# 1. The cluster CA — 10-year self-signed root.
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=kubernetes" -out ca.crt

# 2. The apiserver serving cert. Extensions MUST come from a config file.
cat > apiserver-openssl.cnf <<'EOF'
[req]
distinguished_name = dn
req_extensions     = v3_req
prompt             = no
[dn]
CN = kube-apiserver
[v3_req]
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName   = @alt
[alt]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = cp1
IP.1  = 10.96.0.1          # first IP of the service CIDR — the in-cluster ClusterIP
IP.2  = 10.10.0.11         # this node's advertise address
IP.3  = 10.10.0.100        # the HA VIP  <-- forget this and you rebuild the cert later
EOF

openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -out apiserver.csr -config apiserver-openssl.cnf
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out apiserver.crt -days 365 -sha256 \
  -extensions v3_req -extfile apiserver-openssl.cnf
```

```bash
# 3. A client cert — note the ONLY differences: EKU, the subject, and no SANs.
#    This one is the apiserver's identity to etcd, so it is signed by the ETCD CA.
cat > etcd-client.cnf <<'EOF'
[req]
distinguished_name = dn
req_extensions     = v3_req
prompt             = no
[dn]
CN = kube-apiserver-etcd-client
[v3_req]
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
openssl genrsa -out apiserver-etcd-client.key 2048
openssl req -new -key apiserver-etcd-client.key -out apiserver-etcd-client.csr -config etcd-client.cnf
openssl x509 -req -in apiserver-etcd-client.csr \
  -CA etcd/ca.crt -CAkey etcd/ca.key -CAcreateserial \
  -out apiserver-etcd-client.crt -days 365 -sha256 \
  -extensions v3_req -extfile etcd-client.cnf
```

KTHW drives `cfssl` instead, which encodes the same ideas as a JSON profile — `"usages":
["signing","key encipherment","server auth"]` versus `["signing","key encipherment","client
auth"]` — plus a `hosts` array that becomes the SAN set. Same three knobs.

**Always read back what you produced.** This is the single highest-value command in the lesson:

```console
$ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4368892318273412001 (0x3c9e...)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes                        <-- signed by the CLUSTER CA
        Validity
            Not Before: Aug 18 09:14:22 2026 GMT
            Not After : Aug 18 09:19:22 2027 GMT       <-- 365 days. Nothing renews this for you.
        Subject: CN = kube-apiserver                   <-- becomes the username if used as a client
        ...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication          <-- serverAuth ONLY: cannot be a client cert
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Alternative Name:
                DNS:cp1, DNS:kubernetes, DNS:kubernetes.default,
                DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local,
                IP Address:10.96.0.1, IP Address:10.10.0.11, IP Address:10.10.0.100
                                                       ^^^ every address a client may dial
```

Read three blocks and you have diagnosed 80% of bootstrap failures: **Issuer** (right CA?),
**Extended Key Usage** (right direction?), **Subject Alternative Name** (does it cover the address
the client actually used?). A fourth, `Subject`, tells you what identity the cluster will see.

### kubeconfigs are credential envelopes

A kubeconfig is not a connection string. It is a bundle of *(where to connect, what CA to trust,
what identity to present)*. Here is a complete one with every load-bearing field annotated:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: https://10.10.0.100:6443      # MUST be covered by a SAN in apiserver.crt
    certificate-authority-data: LS0tLS1CRUdJTi...   # base64(ca.crt) — what we trust the server to present
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTi...      # base64(admin.crt) — CN=kubernetes-admin
    client-key-data: LS0tLS1CRUdJTi...              # base64(admin.key) — the private key. Guard it.
contexts:
- name: kubernetes-admin@kubernetes
  context:
    cluster: kubernetes
    user: kubernetes-admin
current-context: kubernetes-admin@kubernetes
```

kubeadm writes these under `/etc/kubernetes/`. The identities, straight from
`cmd/kubeadm/app/phases/kubeconfig/kubeconfig.go`:

| File | CN (→ username) | O (→ groups) | Authorized by |
|---|---|---|---|
| `admin.conf` | `kubernetes-admin` | `kubeadm:cluster-admins` | ClusterRoleBinding `kubeadm:cluster-admins` → ClusterRole `cluster-admin` |
| `super-admin.conf` | `kubernetes-super-admin` | `system:masters` | hard-wired in the authorizer — bypasses RBAC entirely |
| `kubelet.conf` | `system:node:<node name>` | `system:nodes` | Node authorizer + RBAC |
| `controller-manager.conf` | `system:kube-controller-manager` | — | built-in ClusterRole of the same name |
| `scheduler.conf` | `system:kube-scheduler` | — | built-in ClusterRole of the same name |

**The `admin.conf` / `super-admin.conf` split is the fact most tutorials still have wrong.** Up to
and including kubeadm v1.28, `admin.conf` carried `O=system:masters`, which meant every copy of
that file was an un-revocable RBAC bypass: `system:masters` is checked *before* RBAC, so you
cannot restrict it, and you cannot revoke a certificate — your only remedy is rotating the cluster
CA. Kubernetes 1.29 (PR #121305) changed this. `admin.conf` now uses the ordinary group
`kubeadm:cluster-admins`, which is bound to `cluster-admin` by an ordinary ClusterRoleBinding you
can edit or delete; the break-glass identity moved into a separate `super-admin.conf` that you are
expected to move off the node to a safe place. `kubeadm join --control-plane` generates only
`admin.conf`, never `super-admin.conf`.

Build one by hand once and it stops being mysterious:

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true \
  --server=https://10.10.0.100:6443 --kubeconfig=admin.conf
kubectl config set-credentials kubernetes-admin \
  --client-certificate=admin.crt --client-key=admin.key --embed-certs=true \
  --kubeconfig=admin.conf
kubectl config set-context kubernetes-admin@kubernetes \
  --cluster=kubernetes --user=kubernetes-admin --kubeconfig=admin.conf
kubectl config use-context kubernetes-admin@kubernetes --kubeconfig=admin.conf
```

### Static pods: how the kubelet breaks the chicken-and-egg

**The problem.** The control-plane components need to run as containers, but the mechanism that
normally runs containers (Deployment → controller-manager → apiserver → etcd) is exactly what does
not exist yet.

**The mechanism.** The kubelet has a second, apiserver-independent source of pods. It watches the
directory named by `staticPodPath` in `KubeletConfiguration` (kubeadm sets
`/etc/kubernetes/manifests`), reads any Pod manifest it finds there, and runs it directly through
the CRI. No scheduler, no API, no etcd. The kubelet re-reads the directory on a timer and on
inotify events, so *editing a file is the deployment mechanism*.

Static pods have three properties that bite people:

1. **They are not managed through the API.** Once the apiserver is up, the kubelet creates a
   read-only **mirror pod** for each one so `kubectl get pods -n kube-system` shows them. Deleting
   the mirror pod does nothing durable: the kubelet recreates it. The manifest file on disk is the
   only source of truth.
2. **The pod name gets the node name appended** — `kube-apiserver-cp1`, `etcd-cp1`. That is how
   you tell three stacked apiservers apart.
3. **They cannot reference Secrets or ConfigMaps**, because those live in the API that does not
   exist yet. Everything is `hostPath` mounts of files on disk. That is why the whole PKI lives
   under `/etc/kubernetes/pki` as plain files.

The bootstrap sequence, in time:

```
  t=0   systemctl start kubelet
        │  kubelet reads /var/lib/kubelet/config.yaml
        │  no /etc/kubernetes/kubelet.conf yet → it cannot register a Node
        │  but staticPodPath is set, so it starts its file-watch loop
        ▼
  t=1   sees /etc/kubernetes/manifests/etcd.yaml
        │  pulls registry.k8s.io/etcd:3.6.8-0 via CRI, starts it
        │  etcd mounts /etc/kubernetes/pki/etcd (hostPath) and /var/lib/etcd
        ▼
  t=2   etcd elects itself leader (single member: quorum of 1)
        ▼
  t=3   sees /etc/kubernetes/manifests/kube-apiserver.yaml
        │  apiserver starts, dials 127.0.0.1:2379 presenting apiserver-etcd-client.crt
        │  ── if the etcd CA is wrong, THIS is where you crash-loop ──
        ▼
  t=4   apiserver serves /readyz → 200
        ▼
  t=5   kubelet's own client credential (kubelet.conf) now works
        │  → kubelet registers the Node object
        │  → apiserver creates MIRROR PODS for etcd + apiserver + CM + scheduler
        ▼
  t=6   controller-manager and scheduler come up, acquire their Leases,
        and the cluster is a cluster.  Node stays NotReady until a CNI lands.
```

In pure KTHW you run the components as **systemd units** instead. Same idea one layer lower: the
init system is the thing that can start a process without an orchestrator. Doing both passes is
the point of the Practice section — you will viscerally understand that "static pod" means
"systemd unit, but the kubelet is the init system."

Here is the load-bearing part of a real `etcd.yaml` static-pod manifest as kubeadm writes it, with
the flags annotated (the flag values are taken from
`cmd/kubeadm/app/phases/etcd/local.go`, k8s 1.36):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  hostNetwork: true              # no CNI exists yet; must use the host's network namespace
  priorityClassName: system-node-critical
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.6.8-0     # kubeadm 1.34–1.36 default; 1.33 shipped 3.5.24
    command:
    - etcd
    - --name=cp1                            # member name == node name (KCP relies on this)
    - --data-dir=/var/lib/etcd
    - --listen-client-urls=https://127.0.0.1:2379,https://10.10.0.11:2379
    - --advertise-client-urls=https://10.10.0.11:2379
    - --listen-peer-urls=https://10.10.0.11:2380
    - --initial-advertise-peer-urls=https://10.10.0.11:2380
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --client-cert-auth=true               # REQUIRE a client cert from the apiserver
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --peer-client-cert-auth=true
    - --snapshot-count=10000                # Raft-log snapshot cadence; see 10.2
    - --listen-metrics-urls=http://127.0.0.1:2381   # plain HTTP, loopback: probes + Prometheus
    - --feature-gates=InitialCorruptCheck=true
    - --watch-progress-notify-interval=5s   # feeds the apiserver's consistent-read-from-cache path
    volumeMounts:
    - {name: etcd-data,  mountPath: /var/lib/etcd}
    - {name: etcd-certs, mountPath: /etc/kubernetes/pki/etcd}
  volumes:
  - {name: etcd-data,  hostPath: {path: /var/lib/etcd, type: DirectoryOrCreate}}
  - {name: etcd-certs, hostPath: {path: /etc/kubernetes/pki/etcd, type: DirectoryOrCreate}}
```

Note what kubeadm does **not** set: no `--quota-backend-bytes` (so you inherit etcd's 2 GiB
default), no `--auto-compaction-*` (the apiserver drives compaction instead), no
`--election-timeout` override. Lesson 10.2 is entirely about what those omissions mean once real
traffic arrives.

### The Node authorizer and NodeRestriction: why the kubelet's CN is load-bearing

A kubelet presents `CN=system:node:cp1, O=system:nodes`. Two mechanisms read that identity:

- **The Node authorizer** (`--authorization-mode=Node,RBAC`) parses the node name straight out of
  the username prefix `system:node:`. It then permits reads only of objects reachable from *that
  node's* pods — the Secrets and ConfigMaps those pods mount, the PVs they bind, its own Node
  object, its own Leases and CSRs — by walking a graph the apiserver maintains. A cert with a
  correct-looking CN for a different node authorizes nothing on this node.
- **The NodeRestriction admission plugin** (`--enable-admission-plugins=NodeRestriction`) then
  constrains *writes*: a kubelet may only modify its own Node and Pod status, may not delete its
  Node, and may not set labels in restricted prefixes (notably
  `node-restriction.kubernetes.io/*`) that could pull privileged workloads onto a compromised
  machine.

Consequence you must internalise: **the CN must be exactly `system:node:<nodename>` and the group
exactly `system:nodes`.** Get the CN wrong and the kubelet authenticates fine and is authorized
for nothing — a confusing state where logs show a valid user and constant 403s. Sign a kubelet
cert with `O=system:masters` (people do this to "make it work") and you have handed every node
cluster-admin.

### Bootstrap tokens and TLS bootstrapping

**The problem.** A worker needs a client certificate to talk to the apiserver, but obtaining one
normally requires talking to the apiserver. Hand-copying a signed key to each of 200 GPU nodes is
not a plan — it is also a security disaster, because you have to move private keys over the
network.

**The mechanism.** A short-lived shared secret authenticates the node just long enough for it to
ask for its own long-lived certificate.

A bootstrap token is a string of the form `[a-z0-9]{6}.[a-z0-9]{16}` — a public **Token ID** and a
secret half. It is stored as a Secret, and the shape matters because you will read one during the
Practice:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-07401b          # name MUST be bootstrap-token-<token id>
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: "07401b"                    # the public half
  token-secret: "f395accd246ae52d"      # the secret half
  expiration: "2026-08-19T09:14:22Z"    # kubeadm default TTL: 24h (DefaultTokenDuration)
  usage-bootstrap-authentication: "true"  # may be used as a bearer token
  usage-bootstrap-signing: "true"         # may sign the cluster-info ConfigMap
  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"
```

Presenting `07401b.f395accd246ae52d` as a bearer token authenticates you as the user
`system:bootstrap:07401b` in the group `system:bootstrappers` plus whatever
`auth-extra-groups` lists. That identity can do almost nothing — which is the point.

kubeadm creates exactly four RBAC objects to make the flow work (from
`cmd/kubeadm/app/phases/bootstraptoken/node/tlsbootstrap.go`):

| Object | Binds | To | So that |
|---|---|---|---|
| CRB `kubeadm:kubelet-bootstrap` | group `system:bootstrappers:kubeadm:default-node-token` | ClusterRole `system:node-bootstrapper` | a bootstrapping node can **create** CSRs |
| CRB `kubeadm:get-nodes` | same group | ClusterRole `kubeadm:get-nodes` (`get nodes`) | `kubeadm join` can check the node doesn't already exist |
| CRB `kubeadm:node-autoapprove-bootstrap` | same group | ClusterRole `system:certificates.k8s.io:certificatesigningrequests:nodeclient` | the csrapprover **auto-approves** the first CSR |
| CRB `kubeadm:node-autoapprove-certificate-rotation` | group `system:nodes` | ClusterRole `…:selfnodeclient` | already-joined nodes can **auto-renew** their own client cert |

And the sequence, as messages:

```
  WORKER (kubelet)                                  APISERVER            CONTROLLER-MANAGER
        │                                               │                  (csrapprover +
        │  1. GET /api/v1/namespaces/kube-public/       │                   csrsigner)
        │     configmaps/cluster-info    (anonymous)    │                        │
        │ ─────────────────────────────────────────────▶│                        │
        │ ◀──── ca.crt + apiserver URL, JWS-signed ─────│                        │
        │       verify signature with the token secret, │                        │
        │       AND/OR verify sha256 of the CA public   │                        │
        │       key against --discovery-token-ca-cert-  │                        │
        │       hash.  ── this is the step that stops   │                        │
        │       a MITM from handing you a fake CA ──    │                        │
        │                                               │                        │
        │  2. writes /etc/kubernetes/bootstrap-kubelet.conf                      │
        │     (identity = the bootstrap token)          │                        │
        │                                               │                        │
        │  3. generates a keypair, POSTs a CSR:         │                        │
        │     CN=system:node:worker-7, O=system:nodes   │                        │
        │     signerName=kubernetes.io/kube-apiserver-  │                        │
        │                client-kubelet                 │                        │
        │ ─────────────────────────────────────────────▶│                        │
        │                                               │ ──── watch event ─────▶│
        │                                               │                        │ 4. approve:
        │                                               │                        │    subject is in
        │                                               │                        │    system:bootstrappers:
        │                                               │                        │    kubeadm:default-node-token
        │                                               │                        │    and CN matches the
        │                                               │                        │    requesting user
        │                                               │◀─── Approved ──────────│
        │                                               │                        │ 5. sign with ca.key
        │                                               │◀─── status.certificate │
        │  6. GET csr → downloads the signed cert       │                        │
        │ ◀─────────────────────────────────────────────│                        │
        │     writes /var/lib/kubelet/pki/kubelet-client-current.pem             │
        │     writes /etc/kubernetes/kubelet.conf pointing at it                 │
        │                                               │                        │
        │  7. from now on: rotation. At ~70–90% of the cert's lifetime the       │
        │     kubelet POSTs a new CSR signed with its CURRENT cert; the          │
        │     selfnodeclient binding auto-approves it. No token involved.        │
```

Two signers, and knowing the difference is a real interview discriminator:

- `kubernetes.io/kube-apiserver-client-kubelet` — the kubelet's **client** cert (identity to the
  apiserver). Auto-approved by the above bindings, auto-rotated by the kubelet by default.
- `kubernetes.io/kubelet-serving` — the kubelet's **serving** cert (presented when the apiserver
  dials in for `logs`/`exec`). Requires `serverTLSBootstrap: true` in `KubeletConfiguration`, and
  **is deliberately not auto-approved** by any built-in controller, because the apiserver cannot
  verify that a node really owns the IP it puts in the SAN. You approve these yourself or run an
  approver. This is why "certs auto-rotate" is only half true, and why a cluster can lose
  `kubectl logs` a year in while `kubectl get` keeps working.

### HA endpoint: `controlPlaneEndpoint` and why it must be decided first

With more than one control-plane node, clients dial a **stable endpoint** — a VIP or a DNS name —
which a load balancer fans out to the live apiservers on `:6443`. Two constraints are yours:

- **The endpoint must be in every apiserver serving cert's SAN list.** SANs are fixed at
  generation time. `kubeadm certs renew` re-signs the *same* fields; it does not discover new ones.
  Retrofitting a VIP means regenerating `apiserver.crt` (delete `apiserver.crt`/`apiserver.key`,
  re-run `kubeadm init phase certs apiserver` with the endpoint set, restart the static pod) —
  doable, but you will do it in an outage window rather than at your leisure.
- **The load balancer must do L4 TCP passthrough.** mTLS is end-to-end to the apiserver: if the LB
  terminates TLS, the client certificate never reaches the apiserver and every request
  authenticates as anonymous.

`kubeadm init --control-plane-endpoint=<vip>:6443` bakes it in. **Set it even on a single-node
cluster you might grow later** — pointing it at that node's own IP costs nothing and preserves
your options. Lesson 10.3 builds the VIP itself.

### Topology: stacked vs external etcd

Two shapes, and the choice changes your PKI and your blast radius:

- **Stacked** — etcd runs as a static pod *on each control-plane node*, colocated with the
  apiserver, and the apiserver talks to it over loopback. Fewest machines. Losing a control-plane
  node loses an etcd member too, so a 3-node stacked cluster tolerates exactly one node failure.
- **External** — etcd is its own cluster on its own machines with its own fast disks. The
  apiserver reaches it purely as a client. More hardware and a second thing to patch and back up,
  but the etcd failure domain and the apiserver failure domain are decoupled.

The PKI consequence: in the **external** case you typically do not have the etcd CA key on the
control-plane nodes at all — you are handed `apiserver-etcd-client.crt/.key` plus `etcd/ca.crt`
and configure `external.caFile/certFile/keyFile` in `ClusterConfiguration`. In the **stacked**
case kubeadm generates and holds the etcd CA itself. Either way the apiserver→etcd credential is
a client cert **signed by the etcd CA**; that fact is topology-independent, and it is the single
most common hand-build error.

### Certificate lifetime, and why it fails slowly rather than all at once

kubeadm's constants (`cmd/kubeadm/app/constants/constants.go`, k8s 1.36):

| Constant | Value | Configurable via |
|---|---|---|
| `CertificateValidityPeriod` (all leaves) | 365 days | `ClusterConfiguration.certificateValidityPeriod` (v1beta4) |
| `CACertificateValidityPeriod` (all CAs) | 3650 days (10 y) | `ClusterConfiguration.caCertificateValidityPeriod` (v1beta4) |
| `DefaultCertTokenDuration` (the `--certificate-key` for `join --control-plane`) | 2 hours | `--certificate-key` / re-run `kubeadm init phase upload-certs` |
| `DefaultTokenDuration` (bootstrap token) | 24 hours | `--token-ttl` |

Renewal is **not** automatic for the control-plane leaves. `kubeadm upgrade apply` renews them as
a side effect, which is why clusters that get upgraded regularly never see this problem and
clusters that are left alone for a year die. `kubeadm certs check-expiration` reports the state:

```console
$ kubeadm certs check-expiration
CERTIFICATE                ...  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                      41d             ca                      no
apiserver                       41d             ca                      no
apiserver-etcd-client           41d             etcd-ca                  no
apiserver-kubelet-client        41d             ca                      no
controller-manager.conf         41d             ca                      no
etcd-healthcheck-client         41d             etcd-ca                  no
etcd-peer                       41d             etcd-ca                  no
etcd-server                     41d             etcd-ca                  no
front-proxy-client              41d             front-proxy-ca           no
scheduler.conf                  41d             ca                      no
super-admin.conf                41d             ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Aug 16, 2035 09:14 UTC   3285d           no
etcd-ca                 Aug 16, 2035 09:14 UTC   3285d           no
front-proxy-ca          Aug 16, 2035 09:14 UTC   3285d           no
```

Read it as three questions. Which certs are close to expiry? Which CA signed each — that column is
a free confirmation that `apiserver-etcd-client` is signed by `etcd-ca` and not `ca`. And is
anything `EXTERNALLY MANAGED` (meaning the key is absent and kubeadm cannot renew it — normal for
external-CA setups, alarming otherwise)? Note `kubelet.conf` is usually absent from this list on
nodes where the kubelet auto-rotates: kubeadm points `kubelet.conf` at
`/var/lib/kubelet/pki/kubelet-client-current.pem` rather than embedding the cert.

**The sharp edge for HA clusters built incrementally.** Each leaf's 365-day clock starts the day
it is minted. Control-plane nodes are typically added on different days — the initial bootstrap,
a third node added a month later, a node re-joined after a mainboard swap. Their certs therefore
expire on *different days*, so what you get is not one clean outage but a drip: one etcd peer
cert first (quorum degrades from 3 to 2 and nobody notices), then an apiserver serving cert weeks
later (one third of `kubectl` calls fail depending on which backend the LB picks), then another.
**A partially-broken cluster is harder to diagnose than a completely broken one**, because every
symptom is intermittent. Check every node, not the one you built first.

`kubeadm certs renew all` re-signs the leaves; you must then restart the static pods (moving the
manifests out of and back into `/etc/kubernetes/manifests`, or `crictl rm` the containers) because
the components read their certs once at startup. Put `check-expiration` in a monthly cron on every
control-plane node with an alert threshold of 30 days. It is the cheapest outage you will ever
prevent.

At telco/edge scale, PKI ownership becomes a first-class platform concern rather than a cron job.
Deutsche Telekom's ["Das Schiff"](https://github.com/telekom/das-schiff) CaaS platform runs Cluster
API + Metal3 (+ vSphere) + Flux across hundreds of bare-metal locations in Germany, engineered so
clusters keep running on correctly-issued certs and reconciled GitOps state even when the
management plane is unreachable. Same problem, automated; lesson 10.4 covers the machinery.

### Encryption at rest — a bare-metal concern EKS hid

On a managed control plane the provider encrypted etcd's disk for you, usually with a KMS key. On
bare metal, **Secrets sit in etcd as base64 — that is, in plaintext — by default.** Anyone with
the etcd data directory, or a snapshot copied off the node, can read every credential in the
cluster. Since you are about to start taking etcd snapshots and shipping them to object storage
(lesson 10.2), this is not theoretical.

You wire it yourself with an `EncryptionConfiguration` passed as
`--encryption-provider-config=/etc/kubernetes/enc/enc.yaml`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets", "configmaps"]
    providers:
      - aescbc:                      # first provider = the one used to ENCRYPT writes
          keys:
            - name: key1
              secret: <base64 of 32 random bytes>
      - identity: {}                 # last = read plaintext, for the migration window
```

**Order is the whole mechanism.** The first entry encrypts new writes; every entry can decrypt.
Keeping `identity` last lets the apiserver still read objects written before you turned this on.
Once you have rewritten everything —

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

— every object has been re-persisted through the new provider, and you can drop `identity` to make
plaintext unreadable. Reverse the order (`identity` first) and you have silently turned encryption
off while believing it is on. For production prefer a `kms` provider so the key is not sitting in
a file next to the data it protects. Add the operational cost of key management to the
capex-vs-cloud model in 10.8 — it used to be in the managed-control-plane fee.

### `kubeadm init` phases — this IS your diff

`kubeadm init` is a pipeline of named phases, each runnable on its own via `kubeadm init phase
<name>`. The order below is read from `initRunner.AppendPhase(...)` in
`cmd/kubeadm/app/cmd/init.go` (k8s 1.36) — several widely-copied blog tables get it wrong, notably
by putting `control-plane` before `etcd`:

| # | Phase | What it does | Your hand-built equivalent |
|---|---|---|---|
| 1 | `preflight` | port/CPU/swap/CRI checks, image pulls | reading the error messages you would otherwise generate yourself |
| 2 | `certs` | the whole PKI: 3 CAs + 8 leaves + `sa` keypair | your `openssl`/`cfssl` run |
| 3 | `kubeconfig` | `admin.conf`, `super-admin.conf`, `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` | your `kubectl config set-*` runs |
| 4 | `etcd` | writes `etcd.yaml` static pod (local etcd only) | your etcd systemd unit |
| 5 | `control-plane` | writes `kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml` | your three systemd units |
| 6 | `kubelet-start` | writes kubelet config + starts the kubelet | `systemctl start kubelet` |
| 7 | `wait-control-plane` | polls the apiserver until it is healthy | you, watching logs |
| 8 | `upload-config` | stores `ClusterConfiguration`/`KubeletConfiguration` as ConfigMaps | nothing — you have no config to share |
| 9 | `upload-certs` | encrypts the CA keys into the `kubeadm-certs` Secret (2 h TTL) | `scp` (and this is why you should not) |
| 10 | `mark-control-plane` | labels + taints the node | `kubectl label` / `kubectl taint` |
| 11 | `bootstrap-token` | creates the token + the four RBAC objects above | your hand-written RBAC |
| 12 | `kubelet-finalize` | points `kubelet.conf` at the rotating cert | manual edit |
| 13 | `addon` | CoreDNS + kube-proxy | your addon manifests |
| 14 | `show-join-command` | prints the `kubeadm join` line | you, assembling it |

`kubeadm join` runs a parallel pipeline: `preflight` → `control-plane-prepare` → `check-etcd` →
`kubelet-start` → `etcd-join` → `kubelet-wait-bootstrap` → `control-plane-join` →
`wait-control-plane`. Note `check-etcd` *before* `etcd-join`: a control-plane join refuses to add
an etcd member unless the existing cluster is healthy, because adding a member to an unhealthy
quorum is how you turn a degraded cluster into a dead one (10.2 and 10.3 both come back to this).

The fastest way to produce your diff table:

```bash
kubeadm init phase certs all --dry-run --control-plane-endpoint 10.10.0.100:6443 \
  | tee kubeadm-certs-dryrun.txt
kubeadm config print init-defaults --component-configs KubeletConfiguration
```

### Bootstrap error → cert at fault (field cheat-sheet)

Almost every bootstrap failure is one cert, one SAN, or one EKU. Memorise the mapping:

| Error you see | Where | Cause |
|---|---|---|
| `x509: certificate is valid for X, not Y` | any client | serving cert **missing a SAN** for the address dialled — usually the VIP or a node IP |
| `x509: certificate signed by unknown authority` | apiserver → etcd | `apiserver-etcd-client` signed by the **cluster CA** instead of the **etcd CA** |
| `x509: certificate signed by unknown authority` | kubectl → apiserver | `certificate-authority-data` in the kubeconfig is not the CA that signed `apiserver.crt` |
| `remote error: tls: bad certificate` | apiserver → etcd | cert/key mismatch, or etcd's `--trusted-ca-file` is not the CA that signed your client cert |
| `x509: certificate specifies an incompatible key usage` | anywhere | EKU wrong: client cert minted with `serverAuth`, or vice versa |
| `Unauthorized` | kubelet → apiserver | kubelet cert CN is not `system:node:<name>`, or `--client-ca-file` on the apiserver is the wrong CA |
| `403 Forbidden` on `kubectl logs` | apiserver → kubelet | ClusterRoleBinding `kubeadm:apiserver-kubelet-client` missing, or kubelet's `--authorization-mode` is `AlwaysAllow`/misconfigured |
| SA tokens rejected on one apiserver only | workload | `sa.pub` on that node does not match the `sa.key` used to sign |
| aggregated API returns `401` | metrics-server etc. | `front-proxy-client` or `--requestheader-*` misconfigured |
| `Unable to register node ... nodes is forbidden` | kubelet | NodeRestriction rejecting a mismatched node name |

Reach for `openssl x509 -noout -text` on the named cert first. It is faster than reading apiserver
logs blind, and three fields (Issuer, EKU, SAN) resolve most of the table.

### Provisioning arithmetic: how long does a fleet take?

Bare-metal provisioning is a pipeline with a fixed per-node cost and a parallelism ceiling, so
capacity planning is arithmetic rather than intuition. State the model, then run it.

Let:

- `N` = number of nodes,
- `t` = wall-clock time to take one node from power-on to `Ready`,
- `P` = how many nodes you can provision concurrently,
- `S` = the serial part that cannot overlap (control-plane bring-up before any worker can join).

Then total time `T ≈ S + t × ceil(N / P)`.

Work it for a 64-GPU fleet — 8 nodes of 8 GPUs, the same fleet the module's capstone models:

```
  Per-node budget t (measured, not guessed — instrument each stage):
    BMC power-on + POST on a big dual-socket box with lots of DIMMs   4.0 min
    PXE/HTTP boot of kernel+initrd                                    0.5 min
    stream + write a 2.5 GiB OS image to NVMe                         1.5 min
    first boot, cloud-init/machine-config, containerd up              2.0 min
    kubeadm join (or CAPI bootstrap): CSR + approval + kubelet ready  1.5 min
    CNI + node-level DaemonSets settle                                1.0 min
    GPU driver + fabric-manager + DCGM up, node Allocatable shows GPUs 6.0 min
                                                              t =    16.5 min

  Serial prefix S: 3 control-plane nodes brought up first, and the
  first one must finish before the other two can join.
    S = t(cp1) + t(cp2, cp3 in parallel) ≈ 12 + 12 = 24 min
      (control-plane nodes skip the GPU stage, so their t is lower)
```

Now the parallelism term. Three real ceilings, and the smallest one wins:

- **Image server bandwidth.** A 2.5 GiB image at a 10 Gb/s uplink = 2.5 GiB × 8 ÷ 10 Gb/s ≈ 2.0 s
  of pure transfer per node, so bandwidth is *not* your limit at this size — but at 40 GiB
  container-image preloads it becomes one. Compute it, do not assume.
- **Ironic/Tinkerbell worker concurrency.** Ironic's `[conductor] max_concurrent_deploy` bounds
  simultaneous deployments; the default is small enough to surprise people at fleet scale.
- **Rack power and PDU headroom.** Eight 6 kW nodes drawing inrush simultaneously is a real
  breaker conversation. Staggering power-on is often the actual constraint.

With `P = 4`:

```
  T = S + t × ceil(N/P)
    = 24 min + 16.5 min × ceil(8/4)
    = 24 + 33 = 57 min for the whole 64-GPU cluster
```

With `P = 8` (fully parallel) it is `24 + 16.5 = 40.5 min` — a 29% saving. With `P = 1` (a serial
runbook and one engineer) it is `24 + 132 = 156 min`, and that is the number that makes the case
for lesson 10.4's declarative fleet: **the difference between `P = 1` and `P = 8` is entirely a
tooling decision, not a hardware one.**

Two things this model teaches that a table of numbers would not:

1. **Optimise `t`'s largest term first.** Here it is the 6-minute GPU stack, not the 1.5-minute
   image write. Baking drivers into the image (or into a Talos Image Factory schematic, 10.4)
   converts 6 minutes of per-node runtime into zero, and it does so for every node, forever.
2. **`S` does not amortise.** Serial prefixes are pure loss at any `N`. This is the mathematical
   argument for a long-lived management cluster (10.4) that already exists when you provision
   fleet #2.

Scale it up to check the intuition: at `N = 200` with `t = 16.5` and `P = 8`,
`T = 24 + 16.5 × 25 = 436.5 min ≈ 7.3 hours`. That is one working day for a full fleet rebuild —
which is exactly the number you need when someone asks "how long to re-image the fleet for a
security patch?" and the honest answer determines whether you do it.

## Perspectives

**The developer/consumer view.** You have spent a career on the other side of this: `kubectl`
"just worked" because someone else's control plane presented a cert your client already trusted.
That someone is now you. The payoff is that app-level auth bugs — a workload's ServiceAccount
token silently rejected, a webhook failing TLS verification, an aggregated API returning 401 —
stop being mysterious once you can name which cert, which CA, and which SAN is in play.

**The operator view.** Your job is no longer "does the cluster work" but "will the cluster still
work in 11 months." Cert lifetime, SAN completeness for every future VIP, `sa` keypair
consistency across nodes, and a monthly `check-expiration` cron are yours now. This is boring,
unglamorous work, and it is exactly the work that silently protects uptime — which is why it shows
up in interview probes.

**The security/PKI view.** Three independent CAs is a deliberate blast-radius decision, and the
`admin.conf`/`super-admin.conf` split is the same instinct applied to humans: `system:masters`
bypasses RBAC and cannot be revoked, so it belongs in a file you keep off the machine, not in the
kubeconfig everybody copies to their laptop. Kubernetes has no certificate revocation. Rotation
and short lifetimes are the only levers you have, so design as though every private key you mint
is valid until its `notAfter`, because it is.

**The hardware view.** Stages 0–4 of the pipeline are where bare metal differs from everything
else: BMC reachability, boot mode, firmware levels, disk selectors that must match the right
device across heterogeneous chassis. None of it has an API in the cloud sense, all of it is
per-vendor, and all of it becomes a Metal3/Tinkerbell controller's problem in 10.4 and 10.5.

**The economics view.** Every PKI and encryption-at-rest responsibility here was previously bundled
into your managed-control-plane bill. When you build the capex-vs-cloud model in 10.8,
"self-managed control plane" needs a line item for the *engineering time* to run this cron, rotate
these certs, and own this key management — not just hardware capex. And the provisioning
arithmetic above is a direct input: a fleet you can only rebuild at `P = 1` has a hidden cost
measured in idle GPU-hours every time you touch it.

## Real-world use cases

- **"When Kubernetes Certificates Expire: A Production War Story"** —
  <https://medium.com/@olanipekunadekunleoluwole/when-kubernetes-certificates-expire-a-production-war-story-3bd4a54db3bf>
  — a self-managed cluster's apiserver certificate hit the 1-year kubeadm default, expired, and
  froze every deployment while `kubectl` died cluster-wide. The mechanism is exactly the one this
  lesson describes: the leaf's `notAfter` passed, the apiserver's TLS handshake started failing,
  and nothing self-healed because renewal itself requires a working API. Shows why
  `check-expiration` belongs in cron and not in your memory.
- **"2019-12 K8s certificate expiration outage"** —
  <https://vadosware.io/post/2019-12-k8s-cert-expiration-outage/> — an independent postmortem of
  the same failure class with a full diagnosis walkthrough: start from the `x509` error text, work
  backwards to which cert, confirm with `check-expiration`, renew, restart static pods. Pairs
  directly with this lesson's error→cert cheat sheet, generalised from bootstrap-time to
  steady-state ops — the same table triages a fresh `kubeadm init` and a cluster that has been
  running fine for a year.
- **kubeadm PR #121305 (`admin.conf` / `super-admin.conf` split, Kubernetes 1.29)** —
  <https://github.com/kubernetes/kubernetes/pull/121305> — not an outage but a design change worth
  reading as one. The project concluded that shipping a `system:masters` credential as the default
  admin file was a mistake precisely because that group bypasses RBAC and certificates cannot be
  revoked; the fix was to demote `admin.conf` to an ordinary RBAC-bound group and quarantine the
  break-glass identity in a separate file. If your runbooks still say "copy `admin.conf` to your
  laptop," they are pre-1.29.
- **Deutsche Telekom "Das Schiff"** — <https://github.com/telekom/das-schiff> — CAPI + Metal3
  (+ vSphere) + Flux across hundreds of bare-metal locations in Germany, engineered so clusters
  keep serving on correctly-issued certs and reconciled GitOps state even when the management
  plane is unreachable. PKI/bootstrap ownership at a scale where a `check-expiration` cron does
  not cut it; the full CAPI/Metal3 story is lesson 10.4's.

## Worked example — tracing one apiserver→etcd write

A single `kubectl apply -f pod.yaml` against a hand-built cluster, with the credential in play at
every hop and the command that proves each one.

**Hop 1 — kubectl → apiserver.** kubectl loads `admin.conf`, dials `https://10.10.0.100:6443`, and
the TLS handshake does two independent things. The apiserver presents `apiserver.crt`; kubectl
verifies it chains to the `certificate-authority-data` in the kubeconfig **and** that
`10.10.0.100` appears as an IP SAN. kubectl presents `admin.crt`; the apiserver verifies it chains
to `--client-ca-file=/etc/kubernetes/pki/ca.crt` and extracts `CN=kubernetes-admin`,
`O=kubeadm:cluster-admins`.

Prove it without kubectl:

```console
$ openssl s_client -connect 10.10.0.100:6443 \
    -CAfile /etc/kubernetes/pki/ca.crt \
    -cert  /etc/kubernetes/pki/apiserver-kubelet-client.crt \
    -key   /etc/kubernetes/pki/apiserver-kubelet-client.key </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -ext subjectAltName
subject=CN = kube-apiserver
issuer=CN = kubernetes
X509v3 Subject Alternative Name:
    DNS:cp1, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc,
    DNS:kubernetes.default.svc.cluster.local,
    IP Address:10.96.0.1, IP Address:10.10.0.11, IP Address:10.10.0.100
```

If `10.10.0.100` were absent from that SAN list, every client dialling the VIP would fail with
`x509: certificate is valid for 10.10.0.11, not 10.10.0.100` — and only clients dialling the VIP,
which is why this bug survives a "it works from the node" test.

**Hop 2 — authorization.** `kubeadm:cluster-admins` is bound to `cluster-admin` by an ordinary
ClusterRoleBinding, so RBAC allows. Confirm the binding exists rather than assuming:

```console
$ kubectl get clusterrolebinding kubeadm:cluster-admins -o wide
NAME                     ROLE                        AGE   USERS   GROUPS                   SERVICEACCOUNTS
kubeadm:cluster-admins   ClusterRole/cluster-admin   9d            kubeadm:cluster-admins
```

**Hop 3 — apiserver → etcd.** After admission and validation the apiserver must persist. It opens
a TLS connection to `127.0.0.1:2379` presenting **`apiserver-etcd-client.crt`**, signed by the
**etcd CA**. etcd validates it against `--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt` and,
because `--client-cert-auth=true`, refuses the connection outright if there is no valid client
cert. In the other direction, etcd presents `etcd/server.crt` and the apiserver validates it
against `--etcd-cafile`.

Reproduce the apiserver's own connection by hand — this is the fastest way to isolate "is it the
apiserver or is it etcd?":

```console
$ ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt \
    --key=/etc/kubernetes/pki/apiserver-etcd-client.key \
    get /registry/pods/default/trainer-0 --keys-only
/registry/pods/default/trainer-0
```

Two failure signatures to recognise instantly:

- `x509: certificate signed by unknown authority` here means you signed
  `apiserver-etcd-client.crt` with the **cluster** CA. etcd trusts only `etcd/ca.crt`. Confirm
  with `openssl x509 -in apiserver-etcd-client.crt -noout -issuer` — it must print `CN =
  etcd-ca`.
- `remote error: tls: bad certificate` usually means the cert and key do not belong together.
  Compare moduli: `openssl x509 -noout -modulus -in c.crt | sha256sum` versus `openssl rsa -noout
  -modulus -in c.key | sha256sum`.

**Hop 4 — replication.** etcd writes the key under `/registry/pods/default/trainer-0`, replicates
the Raft entry to peers over connections authenticated with `etcd/peer.crt` (which is why peer
certs need both `serverAuth` and `clientAuth` — a peer connection is a client in one direction and
a server in the other), and acknowledges once a quorum has fsynced it. Lesson 10.2 is that
sentence expanded into a whole chapter.

**Hop 5 — watchers act.** The scheduler, authenticated by `scheduler.conf`
(`CN=system:kube-scheduler`), sees the unscheduled pod and writes a `Binding`. The kubelet on the
target node, authenticated by its own `system:node:worker-7` client cert, sees a pod with its
`spec.nodeName` and starts pulling images. Later, when you run `kubectl logs trainer-0`, the
traffic goes the *other* way across that link: the apiserver dials the kubelet on `:10250`
presenting `apiserver-kubelet-client.crt`, and the kubelet authorizes the user
`kube-apiserver-kubelet-client` via the `kubeadm:apiserver-kubelet-client` binding.

**Every hop above is a distinct certificate in a distinct direction.** Write that sentence on the
first page of your deliverable's cert table.

## Practice

Two passes on **3 cheap VMs** (KVM/multipass, or three `e2-small`/`t3.small`); one machine running
multipass is fine.

**Pass 1 — Kubernetes The Hard Way (hand-built).** Follow KTHW end to end, but produce the
artifact as you go rather than afterwards:

- Generate the CA and every cert with `cfssl` or `openssl`: cluster CA, apiserver serving cert
  (get the SAN list right — service-CIDR first IP, every node IP, the future VIP, and the four
  `kubernetes*` DNS names), `apiserver-kubelet-client`, kubelet client certs, the four kubeconfigs,
  the etcd CA plus server/peer/healthcheck certs and `apiserver-etcd-client`, the front-proxy CA
  and client, and the `sa` keypair.
- Bring up etcd, then apiserver/controller-manager/scheduler as **systemd units**, then join
  workers.
- For every cert record: filename, CN, O, SANs, EKU, signing CA, and **who presents it to whom**.
  That table is the deliverable.
- Deliberately break three things and record the exact error text: sign `apiserver-etcd-client`
  with the cluster CA; omit the VIP from the apiserver SANs and then dial the VIP; mint a kubelet
  cert with `CN=worker-7` instead of `CN=system:node:worker-7`. Each should reproduce a row of the
  cheat sheet above.

**Pass 2 — kubeadm, then diff.** On fresh VMs: install containerd with the `systemd` cgroup driver,
then

```bash
kubeadm init --control-plane-endpoint=10.10.0.100:6443 --pod-network-cidr=10.244.0.0/16 --upload-certs
```

install a CNI, and `kubeadm join` a worker with the printed token. Then diff against Pass 1:

- `ls -R /etc/kubernetes/pki` and `kubeadm certs check-expiration` — map every file back to your
  Pass-1 table; note the 365-day leaves and the `CERTIFICATE AUTHORITY` column.
- `ls /etc/kubernetes/manifests` and `crictl ps` — static pods here, systemd units in Pass 1.
- `kubectl -n kube-system get secret -o name | grep bootstrap-token`, then
  `kubectl -n kube-system get secret bootstrap-token-<id> -o yaml` — compare to the Secret shape
  above.
- `kubectl get csr -w` during the join; then `kubectl get csr <name> -o jsonpath='{.spec.signerName}
  {"\n"}{.spec.username}{"\n"}'` to see `kubernetes.io/kube-apiserver-client-kubelet` and
  `system:bootstrap:<id>`.
- `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text` — compare kubeadm's computed
  SAN list to the one you hand-listed.
- `kubectl get clusterrolebinding kubeadm:cluster-admins kubeadm:apiserver-kubelet-client
  kubeadm:kubelet-bootstrap kubeadm:node-autoapprove-bootstrap -o wide` — the RBAC that makes the
  whole thing work.
- `kubeadm init phase certs all --dry-run` on a scratch host to dump the exact cert set.
- Measure `t` for one node (power-on → `Ready`) and write down your own version of the
  provisioning arithmetic with real numbers.

**Acceptance:** (1) a hand-built control plane where `kubectl get nodes` shows a joined worker
`Ready`; (2) a written **"what kubeadm automated" diff** containing the full cert table, the three
deliberate breakages with their error text, the static-pod-vs-systemd difference, the
bootstrap-token/TLS-bootstrap message sequence, and your measured per-node provisioning time. All
of it feeds the deliverable's KTHW writeup and the 10.8 capex model's "engineering time to run
this yourself" line.

## Common pitfalls

- **Forgetting the LB VIP in the apiserver SAN list before `kubeadm init`.** SANs are fixed at
  generation time and `kubeadm certs renew` re-signs the same fields rather than discovering new
  ones. Retrofitting HA means deleting `apiserver.crt`/`apiserver.key` and re-running the certs
  phase with the endpoint set — an outage-window job. Decide the endpoint first, even for a
  single node.
- **Signing `apiserver-etcd-client` with the cluster CA.** etcd trusts only its own CA, no matter
  how "Kubernetes" both CAs feel. The symptom is `x509: certificate signed by unknown authority`
  on the apiserver→etcd hop, and `openssl x509 -noout -issuer` settles it in one second.
- **Passing `-subj` to `openssl` and expecting SANs or EKU.** Extensions only appear if you pass
  `-extensions`/`-extfile`. A cert that looks right in `-subject` but has an empty
  `X509v3 extensions` block will fail hostname verification and key-usage checks — and the errors
  point at the wrong things.
- **Believing `admin.conf` is `system:masters`.** Since Kubernetes 1.29 it is
  `kubeadm:cluster-admins`, an ordinary RBAC group. If you have been telling people "delete the
  ClusterRoleBinding to lock out admin.conf" and it did nothing, you were on ≤1.28; if you assume
  `admin.conf` still bypasses RBAC, you will be surprised the first time someone restricts it.
- **Assuming cert expiry is someone else's problem, like it was on EKS.** The two postmortems
  above are the same story twice. Also: check *every* control-plane node, because incrementally
  built clusters expire in a drip, not a bang.
- **Confusing kubelet client-cert rotation with serving-cert rotation.** The client cert
  auto-rotates via the `selfnodeclient` auto-approval binding. The **serving** cert requires
  `serverTLSBootstrap: true` and is deliberately never auto-approved by a built-in controller.
  Assuming both are handled leaves `kubectl logs` to break a year later.
- **Letting `sa.key`/`sa.pub` drift between control-plane nodes.** Symptom: ServiceAccount tokens
  intermittently rejected, correlated with which apiserver the load balancer picked. Cause: a
  control-plane node built by hand or re-joined without copying the keypair. Fix: use
  `--upload-certs` / `--certificate-key` so `kubeadm join --control-plane` pulls the real keypair
  from the `kubeadm-certs` Secret.

## Self-check

**(a) Which certificate does the apiserver present to etcd, and which to the kubelet?**
**Answer:** To etcd it presents **`apiserver-etcd-client.crt`** — `CN=kube-apiserver-etcd-client`,
EKU `clientAuth`, signed by the **etcd CA** (etcd trusts only `etcd/ca.crt`). To the kubelet it
presents **`apiserver-kubelet-client.crt`** — `CN=kube-apiserver-kubelet-client`, EKU
`clientAuth`, signed by the **cluster CA**, with no `O` at all; its permissions come from the
ClusterRoleBinding `kubeadm:apiserver-kubelet-client` → ClusterRole `system:kubelet-api-admin`.
The opposite direction on that same link is a different cert entirely: the kubelet presents its
own `CN=system:node:<name>, O=system:nodes` client cert to the apiserver.

**(b) What is inside a bootstrap token, and what is it for?**
**Answer:** A `<6-char id>.<16-char secret>` string, stored as a Secret named
`bootstrap-token-<id>` in `kube-system` with type `bootstrap.kubernetes.io/token`, carrying
`token-id`, `token-secret`, `expiration` (kubeadm default TTL 24 h),
`usage-bootstrap-authentication`, `usage-bootstrap-signing`, and `auth-extra-groups`. Presented as
a bearer token it authenticates as user `system:bootstrap:<id>` in group `system:bootstrappers`
plus the extra groups. Its only job is to let a joining node (i) verify the cluster CA via the
JWS-signed `cluster-info` ConfigMap and (ii) POST a CertificateSigningRequest for its own
`system:node:<name>` client certificate, which the csrapprover auto-approves thanks to the
`kubeadm:node-autoapprove-bootstrap` binding. After that the node uses its own cert and rotates it
itself; the token can expire harmlessly.

**(c) Why are the control-plane components static pods rather than Deployments?**
**Answer:** A Deployment is reconciled by kube-controller-manager, which talks to the apiserver,
which needs etcd — none of which exist at bootstrap. The kubelet is the only component that can
run a container with nothing but a file on disk: it watches `staticPodPath`
(`/etc/kubernetes/manifests`) and drives the CRI directly, no API involved. Once the apiserver is
up, the kubelet creates read-only **mirror pods** so they are visible via `kubectl`, but the
manifest file remains the only source of truth — deleting the mirror pod just makes the kubelet
recreate it. Corollaries: static pods cannot mount Secrets or ConfigMaps (they use `hostPath`),
and their names have the node name appended.

**(d) Why do incrementally-built HA clusters fail with a "slow drip" of cert-expiry incidents
instead of one clean outage?**
**Answer:** Each leaf's 365-day clock starts the day it is minted, and control-plane nodes are
added on different days — initial bootstrap, a third node a month later, a re-join after
maintenance. So their certs expire on different days: an etcd peer cert first (quorum silently
degrades 3→2), an apiserver serving cert weeks later (a third of `kubectl` calls fail depending on
which backend the LB picks), and so on. The cluster looks mostly healthy between events and every
symptom is intermittent, which is harder to diagnose than a total failure. The remedy is
`kubeadm certs check-expiration` on *every* control-plane node, on a schedule, alerting at 30 days.

**(e) A client dialling the VIP gets `x509: certificate is valid for 10.10.0.11, not 10.10.0.100`,
but the same command works from the control-plane node itself. What happened, and what are your
options?**
**Answer:** The apiserver serving cert's SAN list contains the node's advertise address but not
the VIP, because `controlPlaneEndpoint` was not set when the cert was minted. Hostname
verification uses SANs only — `CN` has been ignored for this since Go 1.15 — and an IP literal
must match an **IP** SAN. Options, worst to best: (1) regenerate the cert now — delete
`apiserver.crt`/`apiserver.key`, set `controlPlaneEndpoint` (or add the VIP to
`apiServer.certSANs`), run `kubeadm init phase certs apiserver`, restart the static pod; (2) same
thing but scheduled, since every client dialling the VIP is currently broken; (3) prevention —
always set `controlPlaneEndpoint` at `kubeadm init`, even on a single node. What does *not* work
is `kubeadm certs renew apiserver`, which re-signs the existing field set and will happily hand
you a fresh cert with the same missing SAN.

## Connections & what's next

This lesson gave you the trust roots and the bootstrap mechanism; it deliberately stopped at "one
cert per component, one node at a time." Three threads pick up from here. **10.2 (etcd
operations)** takes the etcd you just stood up and makes you own its disk, its quorum, and its 2am
restore — the PKI here is a hard prerequisite, since you need `apiserver-etcd-client` and the etcd
CA working before you can even reach etcd to break it on purpose. **10.3 (control-plane HA)** grows
the single-node PKI to three nodes behind a VIP, which is exactly why the SAN list and the
`sa` keypair consistency mattered here. **10.4 (declarative fleets: CAPI + Talos)** replaces the
manual cert-minting with a controller — and the reason you will be able to debug that controller is
that you did the job by hand first. The provisioning arithmetic here is also the quantitative case
for 10.4: `P = 1` versus `P = 8` is a tooling choice worth hours per fleet operation.

Next: **[10.2 · etcd operations](02-etcd-operations.md)** — you now know how the apiserver reaches
etcd; the next lesson is about what happens to that etcd once it is yours to run.

## References & further reading

**Primary sources**

- **`kubernetes/kubernetes`, `cmd/kubeadm/` (branch `release-1.36`)** —
  <https://github.com/kubernetes/kubernetes/tree/release-1.36/cmd/kubeadm> — read directly and
  used as the authority for every constant in this lesson: `app/phases/certs/certlist.go` (cert
  definitions, CNs, EKUs, CA relationships), `app/constants/constants.go` (validity periods, token
  TTLs, group names, etcd versions), `app/cmd/init.go` and `app/cmd/join.go` (phase order),
  `app/phases/kubeconfig/kubeconfig.go` (the five kubeconfig identities),
  `app/phases/bootstraptoken/node/tlsbootstrap.go` (the four RBAC objects),
  `app/phases/etcd/local.go` (the etcd static-pod flags),
  `app/util/pkiutil/pki_helpers.go` (`GetAPIServerAltNames`). **Note:** kubernetes.io is
  unreachable from this environment's egress proxy, so the upstream documentation pages were *not*
  read for this lesson; everything version-specific was verified against this source tree instead.
  Read for: reproducing any table here yourself, and for checking whether a claim still holds in a
  later release.
- **kubeadm PR #121305 — separate `super-admin.conf` (Kubernetes 1.29)** —
  <https://github.com/kubernetes/kubernetes/pull/121305> — the change that moved `system:masters`
  out of `admin.conf` into a break-glass file and bound `admin.conf` to the new
  `kubeadm:cluster-admins` group. Confirmed via the release notes in
  `CHANGELOG/CHANGELOG-1.29.md` in the same repo. Read for: why the split exists and what
  `kubeadm upgrade apply` does to an older node.
- **Kubernetes The Hard Way** — <https://github.com/kelseyhightower/kubernetes-the-hard-way> — the
  canonical from-scratch bootstrap; every cert and unit by hand. Read for: doing Pass 1 step by
  step. This lesson's cert table is the kubeadm-flavoured version of what KTHW makes you type.
- **kubeadm PKI, certificate-management and TLS-bootstrapping reference pages on kubernetes.io** —
  `/docs/setup/production-environment/tools/kubeadm/kubeadm-certs/`,
  `/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/`,
  `/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/` — **not relied upon**: these
  pages are blocked by this environment's egress proxy and were not fetched. They are the
  human-readable companions to the source above; consult them when you have unrestricted network
  access, but every fact in this lesson was taken from the source tree rather than from them.
- **cfssl** — <https://github.com/cloudflare/cfssl> — the CA/cert toolkit KTHW uses. Read its
  README once to understand CSR profiles and the `-config`/`-profile` mechanism rather than
  copy-pasting commands.

**Real-world engineering blogs**

- **"When Kubernetes Certificates Expire: A Production War Story"** —
  <https://medium.com/@olanipekunadekunleoluwole/when-kubernetes-certificates-expire-a-production-war-story-3bd4a54db3bf>
  — what it shows: a real cluster-wide freeze from one expired apiserver leaf, and why nothing
  self-heals.
- **"2019-12 K8s certificate expiration outage"** —
  <https://vadosware.io/post/2019-12-k8s-cert-expiration-outage/> — what it shows: an independent
  postmortem of the same failure class with the diagnosis path from `x509` error text to fix.
- **Deutsche Telekom "Das Schiff"** — <https://github.com/telekom/das-schiff> — what it shows:
  PKI/bootstrap ownership generalised to hundreds of bare-metal edge sites, with GitOps autonomy
  when the management cluster is unreachable.

**Deeper dives**

- **`etcd-io/etcd` issue #9785 — client-auth usage required on the etcd server certificate** —
  <https://github.com/etcd-io/etcd/issues/9785> — the reason `etcd/server.crt` carries both
  `serverAuth` and `clientAuth`; kubeadm's source carries a `TODO` referencing it. Read for: why
  "tidying" that EKU breaks etcd startup.
- **`kubeadm init phase --help` and `kubeadm init phase certs all --dry-run`** — the fastest
  primary source of all, since it prints what *your* version does rather than what a blog says a
  version did. Read for: producing the Pass-2 diff table without trusting anyone's table,
  including this one.
