---
lesson: "10.5"
title: "Node provisioning: PXE to Ready"
module: "10"
concept: "Node provisioning: PXE to Ready"
status: not-started
est_time: "7h"
prev: "04-declarative-fleets-capi-talos.md"
next: "06-hardware-health-remediation-rma.md"
artifacts: []
sources: 13
---
# 10.5 · Node provisioning: PXE to Ready

> **Concept.** The netboot-to-Ready pipeline — iPXE → provisioning agent (Ironic/Tinkerbell) → firmware → OS image → cluster join — that lays the metal *beneath* the GPU Operator.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 04 gave you the *object model* for a fleet: `Cluster` / `KubeadmControlPlane` /
`MachineDeployment` / `Machine`, reconciled by Cluster API controllers, with Metal3 or
Tinkerbell as the bare-metal infrastructure providers and Talos as the immutable-OS answer
to configuration drift. That lesson compressed everything below the `Machine` object into
one sentence — a bare-metal provider "owns BMC control, PXE/virtual-media boot, hardware
inspection, and disk imaging." This lesson opens that sentence up into the actual wire
protocol.

You will follow one physical server from the instant a technician racks it and applies
power, through the DHCP packet it broadcasts, the firmware binary it downloads, the
in-memory agent OS it runs, the BIOS settings and firmware versions applied while its disk
is still empty, the image streamed onto that disk, and the `kubeadm join` that makes it a
cluster member — and you will end at the exact handoff line where lesson 04's GPU Operator
takes over. Where lesson 04 was the fleet's *declarative shape*, this lesson is the fleet's
*physical bootstrap sequence*, one host at a time, automated across dozens at once.

## Why this matters

On a managed cloud a node is an API call. On metal — the world CoreWeave, NVIDIA DGX
Cloud, Nebius, Crusoe and every other neocloud actually run in — *you* own everything
below the kubelet, and the gap between "powered on in the DC" and "schedulable GPU node"
is a dozen steps that have to be automated, repeatable and auditable across 40 nodes at a
time, not click-driven per host through a BMC web console.

The failure modes are expensive and quiet. A node that booted with stale NIC firmware
negotiates the wrong link speed and drags an entire 500-GPU all-reduce down to the pace of
its slowest rail. A host that did not get **above-4G decoding** enabled in BIOS cannot map
eight 80 GB HBM apertures into the address space above 4 GB, so the kernel enumerates
GPUs it cannot fully use. A golden image with drivers baked in drifts out of sync with
the GPU Operator's driver version, and now every CUDA bump is a fleet re-image instead of
a rolling DaemonSet update.

There is also an arithmetic reason this matters that has nothing to do with correctness.
A 64-GPU fleet costs roughly $2–3M in capex (you will build that number properly in
lesson 08). Depreciation runs from the day the invoice is paid, not the day the node is
`Ready`. At a 6-year straight-line schedule on a $280K node, every node-day spent
un-provisioned costs `280,000 / (6 × 365) ≈ $128` in depreciation alone, before power,
colocation or the revenue the node could have earned. Turning a rack delivery into
billable capacity in hours instead of weeks is not tidiness — it is the difference between
capex that earns and capex that sits.

This is also a direct interview probe. NVIDIA's *Sr SSE, Kubernetes Node Lifecycle, DGX
Cloud* posting asks for "scalable node provisioning" built on CAPI providers; CoreWeave's
platform roles describe "provisioning bare-metal and virtual clusters with Cluster API…
day-2 lifecycle." The expected answer is not "we use Metal3." It is the seven stages,
what each one can fail at, and what the symptom looks like from the outside.

## What's new here (calibration)

You already know the on-prem primitives — PXE, TFTP, DHCP options 66 and 67, IPMI, BMC
power control. None of that is re-taught as an *idea*. What is new is the level of
resolution:

- **The DHCP conversation option by option**, including the four options (60, 77, 93, 94)
  that decide *which binary* the machine gets, and the one (option 77) whose mishandling
  produces the single most common bare-metal provisioning failure: an infinite iPXE
  chainload loop.
- **UEFI HTTP Boot** as a first-class alternative to TFTP — the `HTTPClient` vendor class,
  architecture types 15/16, and why it removes an entire class of slowness.
- **Redfish as a real REST API**, not a synonym for "modern IPMI": the actual resource
  paths, the actual JSON bodies for a one-time PXE boot and a power cycle, and the
  virtual-media path that lets you skip PXE altogether.
- **Two control models, both in source**: Ironic's node state machine (you author *state*,
  the service derives the steps) versus Tinkerbell's `Workflow` of container Actions (you
  author the *sequence*). Same seven stages, different unit of versioning.
- **Cleaning as a graded operation** — Ironic's `erase_devices_metadata` versus
  `erase_devices`, their default priorities, and what Metal3's two-valued
  `automatedCleaningMode` actually turns on.
- **Provisioning throughput as arithmetic** — how long a rack takes, what the bottleneck
  is, and why it is almost never the BMC.
- **The precise handoff boundary to lesson 04.** Provisioning ends at a `Ready`, labelled,
  *driverless* node. Drivers stay out of the golden image on purpose.

## Core concepts

### 1. The problem: a machine with power and no identity

A freshly racked server has firmware, a MAC address, a BMC on the management network, and
nothing else. It has no operating system, no idea which cluster it belongs to, and no way
to be told — every channel you would normally use to configure a machine assumes the
machine is already configured.

Network boot exists to break that circularity. The bootstrap chain has to satisfy three
constraints simultaneously:

1. **The first thing that runs must fit in firmware.** A NIC option ROM is tens of
   kilobytes and speaks a fixed, minimal protocol set. It cannot speak HTTPS, cannot parse
   YAML, cannot authenticate.
2. **Each stage must be able to fetch a larger, more capable stage.** This is why netboot
   is a *chain*: ROM → iPXE → kernel+initrd → agent → installed OS. Each link buys more
   capability than the one before.
3. **Nothing may depend on per-machine hand configuration.** The only per-machine fact
   available at stage zero is the MAC address (and, on UEFI, a system UUID). Everything
   else has to be derived from it by a server.

Every design decision below falls out of those three constraints.

### 2. The DHCP conversation, option by option

A PXE-capable NIC broadcasts a `DHCPDISCOVER` that is a normal DHCP packet plus a
PXE-specific set of options. The options that matter, with their code numbers — these are
verified against iPXE's `src/include/ipxe/dhcp.h` (iPXE `master`, commit `e6d0a97`, read
2026-08-18):

| Code | Name | Direction | What it carries and why you care |
|---|---|---|---|
| 53 | Message type | both | `DHCPDISCOVER` (1), `DHCPOFFER` (2), `DHCPREQUEST` (3), `DHCPACK` (5). |
| 54 | Server identifier | server | Which DHCP server the client is accepting. Two servers answering → non-deterministic boots. |
| 43 | Vendor-encapsulated options | server | The PXE sub-option space: discovery control (6), boot servers (8), boot menu (9), menu prompt (10). Used by ProxyDHCP setups. |
| **60** | **Vendor class identifier** | client | `PXEClient:Arch:00007:UNDI:003016` for classic PXE, `HTTPClient:Arch:00016:UNDI:...` for UEFI HTTP Boot. **The prefix tells the server which transport the client can use.** |
| 61 | Client identifier | client | Usually hardware type + MAC. |
| **66** | **TFTP server name** | server | The boot server, as a name. Also carried as the BOOTP `siaddr` header field ("next-server"). |
| **67** | **Bootfile name** | server | The file to fetch. Also carried in the BOOTP `file` header field. On UEFI HTTP Boot this is a full `http://…` URL. |
| **77** | **User class identifier** | client | iPXE puts the literal string `iPXE` here. **This is the loop-breaker.** |
| **93** | **Client system architecture** | client | A 16-bit IANA architecture code. Decides which binary the server hands back. |
| 94 | Client network interface identifier | client | UNDI major/minor version. Its presence is part of "is this a netboot client?" |
| 97 | Client machine identifier | client | The system UUID, 17 bytes with a leading type byte of 0. Some ROMs omit it; well-written servers tolerate that. |
| 175 | iPXE encapsulated options | both | iPXE's private option space: priority (0x01), `no-pxedhcp` (0xb0), embedded scriptlet (0x51), TLS trust/cert/key (0x5a–0x5c), SAN settings. |

The architecture codes in option 93 are the ones that decide which file you serve. From
IANA's processor-architecture registry (mirrored in `insomniacslk/dhcp`'s
`iana/archtype.go`, `master`, read 2026-08-18) and cross-checked against iPXE's own enum:

| Value | Meaning | Binary you would typically serve |
|---|---|---|
| 0 (0x0000) | Intel x86 PC — legacy BIOS PXE | `undionly.kpxe` |
| 6 (0x0006) | EFI IA32 | `ipxe.efi` (32-bit build) |
| **7 (0x0007)** | **EFI x86-64** — the overwhelming majority of GPU servers | `ipxe.efi` / `snp.efi` |
| 9 (0x0009) | EFI Byte Code | `ipxe.efi` |
| 10 / 11 | EFI ARM32 / ARM64 | `snp-arm64.efi` |
| **15 / 16** | **EFI x86 / x86-64 boot from HTTP** | an HTTP URL, not a TFTP filename |
| 19 | EFI ARM64 boot from HTTP | an HTTP URL |

Tinkerbell's Smee encodes exactly this mapping in
`smee/internal/dhcp/dhcp.go:ArchToBootFile()`; you can read the whole table there
(`tinkerbell/tinkerbell`, commit `725c33d`, read 2026-08-18). It also encodes the
five-condition test for "is this packet a netboot request at all," which is worth
memorising because it is the first thing to check when a machine silently refuses to
boot: message type is DISCOVER or REQUEST; option 60 is present; option 60 starts with
`PXEClient` or `HTTPClient`; option 93 is present; option 94 is present; option 97 is
either absent or exactly 17 bytes starting with `0x00`.

**Two servers, one conversation.** In most bare-metal setups the machine's *address*
comes from the site DHCP server and its *boot instructions* come from the provisioning
service. Three arrangements exist:

- **Full reservation.** The provisioning service is the DHCP server and hands out both.
  Simple; requires you to own DHCP for that VLAN. (Tinkerbell calls this `--dhcp-mode=reservation`;
  it is the default.)
- **ProxyDHCP.** The provisioning service listens on UDP/4011 and on the broadcast, and
  answers *only* with boot information, leaving addressing to the existing DHCP server. It
  needs layer-2 adjacency or a DHCP relay. (`--dhcp-mode=proxy`.)
- **Auto-proxy.** ProxyDHCP that answers even for machines it has never heard of, serving a
  generic script so the machine can register itself. This is how zero-touch enrolment of a
  brand-new rack works. (`--dhcp-mode=auto-proxy`.)

The choice is a network-politics decision as much as a technical one, and it is worth
making explicitly: "who owns DHCP on the provisioning VLAN" is the question that decides
which of the three you are allowed to build.

### 3. The chainload, and the loop that eats your afternoon

The NIC's PXE ROM is not the software you want driving a provisioning pipeline. It speaks
TFTP and nothing else, has no scripting, no HTTPS, no retry logic worth the name. So the
first thing you serve it is a *better bootloader*: iPXE. iPXE then re-does DHCP from its
own, far more capable stack, and asks for its real instructions.

That second DHCP request is where fleets break. It looks almost identical to the first
one — same MAC, same architecture, same vendor class prefix. If the server answers it the
same way, it serves `ipxe.efi` again. iPXE loads iPXE, which asks again, forever. The
machine console shows a two-line banner scrolling past every few seconds and nothing else
ever happens.

**The fix is option 77.** iPXE unconditionally includes `DHCP_USER_CLASS_ID` with the
four-byte string `iPXE` in every request it sends — this is hard-coded in
`src/net/udp/dhcp.c` in the `dhcp_request_options_data` table, not something you configure.
So the server's rule is:

```
if option 77 == "iPXE"  →  the client is already iPXE   →  serve the SCRIPT
else                    →  the client is a firmware ROM →  serve the BINARY
```

There is a second-order version of this trap. If your provisioning service *also* ships a
customised iPXE build (Tinkerbell patches its embedded binaries to set the user class to
`Tinkerbell`), you have to match on that string as well, or you loop again one level down.
Smee's `Bootfile()` function keeps the three cases in an explicitly ordered switch and the
source comment says the quiet part out loud: *"If a machine is in an ipxe boot loop, it is
likely to be that we aren't matching on IPXE or Tinkerbell userclass (option 77)."*

**UEFI HTTP Boot skips the chainload entirely.** A firmware that advertises option 60 with
the `HTTPClient` prefix and architecture 16 is telling you it can fetch over HTTP by
itself. Serve it a full URL in option 67 and it downloads the next stage directly. The
practical benefit is transport speed, and it is not small: classic TFTP transfers 512-byte
blocks in strict lockstep, one un-acknowledged block at a time, so throughput is bounded
by `block_size / round_trip_time`. At a 1 ms RTT that is `512 B / 0.001 s ≈ 512 KB/s`, and
a 1 MB iPXE binary takes about two seconds of pure protocol overhead per machine. RFC 2348
block-size negotiation and RFC 7440 windowed TFTP improve this substantially when both
ends support them, but plain HTTP over a single TCP connection with a normal congestion
window is simply not in the same class. For the ~1 MB bootloader this is annoying; for a
600 MB initrd it is the difference between a 4-second and a 4-minute stage.

### 4. The boot chain as a message sequence — and what each hop breaks like

```
  MACHINE                DHCP/Smee/dnsmasq        TFTP/HTTP server       Tink/Ironic API
  (NIC ROM → iPXE)          (UDP 67/4011)          (UDP 69 / TCP 80)      (gRPC 42113 / REST)
     │                            │                        │                      │
 ①   │ DHCPDISCOVER ─────────────▶│                        │                      │
     │  opt60 PXEClient:Arch:00007│                        │                      │
     │  opt93 = 0x0007  opt94     │                        │                      │
     │  opt77 ABSENT              │                        │                      │
     │◀─── DHCPOFFER ─────────────│                        │                      │
     │   siaddr = 10.0.0.5        │                        │                      │
     │   opt67 = "ipxe.efi"       │                        │                      │
     │                            │                        │                      │
 ②   │ TFTP RRQ ipxe.efi ─────────┼───────────────────────▶│                      │
     │◀──────── 512-byte blocks, lockstep ─────────────────│                      │
     │                            │                        │                      │
 ③   │ [iPXE now running] DHCP ──▶│                        │                      │
     │  opt77 = "iPXE"            │                        │                      │
     │◀─── DHCPACK ───────────────│                        │                      │
     │   opt67 = http://10.0.0.5:7080/ipxe/script/         │                      │
     │              b8:ce:f6:00:07:aa/auto.ipxe            │                      │
     │                            │                        │                      │
 ④   │ GET /ipxe/script/…/auto.ipxe ──────────────────────▶│──── look up MAC ────▶│
     │◀──── "#!ipxe …  kernel …  initrd …  boot" ──────────│◀─── Hardware/BMH ────│
     │                            │                        │                      │
 ⑤   │ GET /vmlinuz-x86_64 ───────┼───────────────────────▶│                      │
     │ GET /initramfs-x86_64 ─────┼───────────────────────▶│  (300–800 MB)        │
     │                            │                        │                      │
 ⑥   │ [agent OS running in RAM] ─┼────────────────────────┼── register agent ───▶│
     │◀───────────────────────────┼────────────────────────┼── inventory / steps ─│
     ▼                            ▼                        ▼                      ▼

  ── WHERE IT BREAKS, AND WHAT YOU SEE ──────────────────────────────────────────────
  ① no DHCPOFFER          → console sits on "Start PXE over IPv4… PXE-E18"; nothing
                             at all appears in the provisioning service's logs, because
                             the packet never reached it. Cause: wrong VLAN, missing
                             `ip helper-address` relay, or the port is still in
                             spanning-tree listening when the ROM gives up. Check the
                             switch first, not the software.
  ① OFFER but wrong file  → machine downloads a binary and instantly resets. Cause:
                             opt93 said 0x0007 (UEFI) and you served undionly.kpxe
                             (BIOS). The symptom is a fast reboot loop with no banner.
  ② TFTP times out        → "PXE-E32: TFTP open timeout". Cause: siaddr points at a
                             host with no TFTP listener, or a firewall drops UDP/69,
                             or the ephemeral-port return path is blocked.
  ③ script never served   → iPXE banner scrolls forever, ~5 s per cycle. Cause: the
                             server is not matching option 77 and is serving the
                             binary again. THE classic bare-metal failure.
  ④ 404 on the script     → "Could not open … (http/404)". Cause: no Hardware/BMH
                             object for this MAC, or the MAC format in the URL path
                             (colon vs dash vs bare) does not match what the service
                             expects.
  ⑤ initrd download stalls→ iPXE prints a byte counter that stops moving. Cause: TFTP
                             for a 600 MB initrd, or an image server saturating its
                             NIC because 40 nodes started at once. See §12.
  ⑥ agent never registers → provisioning service shows the node stuck in `inspecting`
                             / `PREPARING` and eventually times out. Cause: the agent
                             booted but has no route to the API (wrong VLAN once the
                             kernel's own NIC driver took over from the firmware's),
                             or `grpc_authority` in the kernel command line is wrong.
```

Read that failure column as a decision procedure: **the stage that fails tells you which
subsystem owns the bug**, and the stages are ordered by how little they depend on your
software. Everything at ① and ② is network and firmware; ③ and ④ are your provisioning
service's matching logic; ⑤ is transport capacity; ⑥ is the data plane between the
ephemeral OS and your control plane.

### 5. The iPXE script, read line by line

Here is the shape of the script Tinkerbell's Smee generates to boot HookOS. This is
adapted from `smee/internal/ipxe/script/hook.go` — the structure, control flow and kernel
parameters are upstream's; the annotations are mine.

```
#!ipxe

echo Loading the provisioning agent...

set arch      x86_64
set base-url  http://10.0.0.5:7080
set kernel    vmlinuz-${arch}
set initrd    initramfs-${arch}
set retries:int32     3
set retry_delay:int32 5

set idx:int32 0
:retry_kernel
kernel ${base-url}/${kernel} \
  initrd=${initrd} \
  grpc_authority=10.0.0.5:42113 \
  worker_id=${mac} hw_addr=${mac} \
  syslog_host=10.0.0.5 \
  modules=loop,squashfs,sd-mod,usb-storage \
  intel_iommu=on iommu=pt \
  console=tty0 console=ttyS1,115200 \
  && goto download_initrd \
  || iseq ${idx} ${retries} && goto kernel-error \
  || inc idx && echo retry in ${retry_delay}s ; sleep ${retry_delay} ; goto retry_kernel

:download_initrd
set idx:int32 0
:retry_initrd
initrd ${base-url}/${initrd} && goto boot \
  || iseq ${idx} ${retries} && goto initrd-error \
  || inc idx && sleep ${retry_delay} ; goto retry_initrd

:boot
boot || goto boot-error

:kernel-error
echo Failed to load kernel
imgfree
exit
```

Five things in there are load-bearing and none of them are obvious:

- **`${mac}` is an iPXE built-in.** iPXE exposes the booting interface's MAC as a settings
  variable, which is how the machine tells the control plane who it is without anyone
  having configured it. `${buildarch}` is the other one you will reach for; note that it
  is the architecture iPXE was *compiled* for, not necessarily the machine's — Smee's
  static script normalises `i386 → x86_64` and `arm32/arm64 → aarch64` for exactly that
  reason.
- **`initrd=${initrd}` inside the `kernel` line is not redundant.** The `initrd` iPXE
  command stages the file in memory; the `initrd=` kernel *parameter* is how the kernel's
  own early boot code finds it. Omit the parameter and you get a kernel panic —
  "VFS: Unable to mount root fs" — that looks like a disk problem and is not.
- **`intel_iommu=on iommu=pt`** puts the IOMMU in passthrough mode. Lesson 02b.6 covers
  why this matters downstream: with the IOMMU in full translation mode, peer-to-peer DMA
  between an NVMe controller and a GPU BAR needs mappings for both devices, and many
  platforms only deliver the GPUDirect Storage fast path with the IOMMU off or in
  passthrough. Getting this wrong in the *agent's* kernel line costs you nothing; getting
  it wrong in the *installed OS* costs you silent compat-mode fallback later.
- **Two serial consoles.** `console=tty0 console=ttyS1,115200` — the last `console=`
  wins for `/dev/console`, but all of them receive kernel messages. Serial is how you
  debug a machine whose network never came up, and it is why the provisioning service also
  runs a syslog collector (Smee listens on UDP/514 for exactly this).
- **The retry loop is not decoration.** iPXE's `||` operator branches on command failure,
  and the `iseq ${idx} ${retries}` guard is what turns a transient HTTP 503 from an
  overloaded image server into a five-second pause instead of a dead node. When 40 machines
  boot simultaneously (§12), this loop is the only thing between you and a thundering herd.

### 6. The agent ramdisk: an operating system whose whole job is to be thrown away

Stage ⑥ boots a complete Linux system entirely into RAM. The disk has not been touched.
Two implementations dominate:

- **Ironic Python Agent (IPA)** — a Python service in a ramdisk built by
  `ironic-python-agent-builder`. It exposes an HTTP API back to Ironic, publishes a
  hardware inventory, and executes *clean steps* and *deploy steps* that Ironic's hardware
  managers define. The hardware-manager abstraction is the extension point: a vendor ships
  a Python class with `get_clean_steps()` and Ironic calls it.
- **HookOS** — Tinkerbell's LinuxKit-based in-memory OS. It runs `tink-agent`, which
  connects back to Tink Server over gRPC (port 42113 by default) and executes each Action
  in a `Workflow` as a container, via a Docker or containerd runtime running inside the
  ramdisk.

The design constraint they share is that **nothing may persist**. Every decision the agent
makes has to be reconstructible from the control plane on the next boot, because there is
no local state to resume from. That is what makes the whole pipeline re-entrant: crash a
node halfway through cleaning and the next boot starts from the control plane's idea of
where it should be, not from a half-written local file.

Running the installer as containers (Tinkerbell) versus as a monolithic agent (Ironic) is
the visible difference. A Tinkerbell Action is an OCI image plus environment:

```yaml
actions:
  - name: stream-image
    image: quay.io/tinkerbell/actions/image2disk:v1.0.0
    timeoutSeconds: 900
    retries: 2
    env:
      IMG_URL:     "http://10.0.0.5:7080/images/ubuntu-24.04-gpu-base.raw.gz"
      DEST_DISK:   "/dev/disk/by-id/nvme-SAMSUNG_MZ..."
      COMPRESSED:  "true"
  - name: write-cloud-init
    image: quay.io/tinkerbell/actions/writefile:v1.0.0
    env:
      DEST_DISK: "/dev/disk/by-id/nvme-SAMSUNG_MZ...-part2"
      FS_TYPE:   "ext4"
      DEST_PATH: "/etc/cloud/cloud.cfg.d/10-tinkerbell.cfg"
      CONTENTS:  |
        datasource:
          Ec2:
            metadata_urls: ["http://10.0.0.5:7080"]
            strict_id: false
  - name: kexec-into-installed-os
    image: quay.io/tinkerbell/actions/kexec:v1.0.0
    background: true            # the Action cannot report success after kexec
```

Note `background: true` on the last Action. `kexec` replaces the running kernel, so the
agent that launched it will never get to report an exit code. Upstream documents this
field precisely for "Actions that do things like kexec, reboot, or power off a machine"
(`api/v1alpha2/tinkerbell/task.go`). A pipeline that does not know about it will mark
every successful provision as a timeout — which is a wonderful example of the general rule
that **provisioning bugs usually look like success or hang, rarely like an error**.

### 7. Inspection and cleaning — and what "automated cleaning" actually erases

Inspection is the agent enumerating the machine and reporting it upward: CPU model and
count, memory size, every block device with model/serial/WWN/size/rotational, every NIC
with MAC and link state, PCI devices by vendor and class, and the system's own serial
number and UUID. Two things make this worth more than a nice inventory page:

1. **It is your acceptance test.** A node that reports seven `0x10de`-class-`0x0302`
   devices instead of eight has a hardware or BIOS problem *right now*, before you have
   spent an hour imaging it. Diffing the inspection output against the expected SKU
   profile is the single cheapest quality gate in the pipeline.
2. **It resolves `rootDeviceHints`.** You cannot hard-code `/dev/nvme0n1` across a fleet;
   NVMe enumeration order is not stable across boots. Metal3's `RootDeviceHints` therefore
   selects by properties, and the matching semantics differ per field — `model` and
   `vendor` match on *substring*, while `deviceName`, `serialNumber`, `wwn`,
   `wwnWithExtension`, `wwnVendorExtension` and `hctl` must match **exactly**
   (`metal3-io/baremetal-operator`, `apis/metal3.io/v1alpha1/baremetalhost_types.go`,
   commit `f08172e`, read 2026-08-18). `minSizeGigabytes` and `rotational` are the two
   coarse filters, and `rotational: false` plus a size floor is usually enough to
   distinguish an NVMe root from a SATA boot device without pinning serials.

**Cleaning is graded, and the grade matters.** Ironic's `GenericHardwareManager` defines
clean steps with priorities; higher priority runs first. Two are directly relevant
(`openstack/ironic`, `ironic/conf/deploy.py` and `ironic/drivers/modules/agent_base.py`,
commit `d275931`, read 2026-08-18):

| Clean step | Default priority | What it does | Wall-clock cost |
|---|---|---|---|
| `erase_devices_metadata` | 99 | Erases partition tables, filesystem superblocks and RAID metadata. Leaves the data blocks alone. | Seconds |
| `erase_devices` | 10 | Full media erase. Uses ATA Secure Erase or, where supported and enabled, `nvme format` (user-data or crypto mode) rather than a block-by-block overwrite. | Minutes with secure erase; **hours** if it falls back to overwriting a multi-TB device |

Setting either priority to `0` disables it. This is the knob behind Metal3's
`automatedCleaningMode`, which is a two-valued enum — `metadata` (the default) or
`disabled` — and nothing else. **If you read `automatedCleaningMode: metadata` as "wipes
the disks," you are wrong in the direction that matters**: the previous tenant's data
blocks are still on the platter, merely unreferenced. For a multi-tenant neocloud that is
a compliance finding, not a nuance, and the answer is a full-erase clean step configured
on the Ironic side, budgeted for in your provisioning time (§12), not a BMH field.

### 8. Firmware and BIOS as declared state, applied before the OS

Firmware is applied in the agent phase, with the disk empty, for four reasons that are
worth being able to defend individually:

1. **The kernel's view of the machine is a function of firmware state.** Above-4G decoding
   and large-BAR support determine whether the eight GPU BARs can be mapped above the
   32-bit boundary at all. SR-IOV determines whether VFs exist. These are read by the
   kernel at PCI enumeration time — after them is too late.
2. **Firmware flashes require a reboot.** Doing that under a live OS means an outage; doing
   it in a ramdisk means a reboot you were going to do anyway.
3. **Cleaning is destructive** and cannot run under the OS it is about to overwrite.
4. **Determinism.** If every node reaches an identical firmware baseline before imaging,
   the image is the only remaining variable, and "re-image and see if it recurs" becomes a
   valid diagnostic step instead of a coin flip.

Metal3 exposes this as three objects that live next to the `BareMetalHost`:
`HostFirmwareSettings` (a name/value map of BIOS settings, with a `FirmwareSchema` object
describing which are read-only, what their allowed values are, and which require a reset),
`HostFirmwareComponents` (component firmware versions — BMC, BIOS — with a desired
version you can set), and `BareMetalHost.spec.raid` for controller configuration. The
important property is that they are *declarative and reconciled*: the operator reads the
current settings via Redfish, diffs against desired, and schedules the changes into the
next cleaning window.

```yaml
apiVersion: metal3.io/v1alpha1
kind: HostFirmwareSettings
metadata:
  name: gpu-node-07          # same name as the BareMetalHost, by convention
  namespace: metal3
spec:
  settings:
    BootMode:                  "Uefi"
    SecureBoot:                "Disabled"      # NVIDIA driver signing story, lesson 04
    Above4GDecoding:           "Enabled"       # ← eight 80 GB BARs above the 4 GB line
    ResizableBarSupport:       "Enabled"
    SriovGlobalEnable:         "Enabled"
    ProcVirtualization:        "Enabled"
    IommuSupport:              "Enabled"
    NumLock:                   "Off"
    SerialPortAddress:         "Serial1Com2"   # matches console=ttyS1 in the iPXE script
```

Setting names are vendor-specific — that YAML is Dell-flavoured; on a Supermicro or HPE
board the same semantics live under different keys. **Read `FirmwareSchema` for the host
in front of you rather than copying a manifest between vendors**; a name that does not
exist in the schema is silently not applied, which is the quietest failure in this lesson.

### 9. Redfish, concretely

Redfish is a REST API over HTTPS with a JSON hypermedia data model rooted at
`/redfish/v1/`. Three operations carry almost all of provisioning. The enum values below
are verified against `openstack/sushy` (`sushy/resources/system/constants.py` and
`sushy/resources/constants.py`, commit `3e7bd47`, read 2026-08-18), which is the Redfish
client Ironic drives.

**Read the current boot configuration:**

```
GET /redfish/v1/Systems/System.Embedded.1
```
```json
{
  "@odata.id": "/redfish/v1/Systems/System.Embedded.1",
  "Id": "System.Embedded.1",
  "PowerState": "Off",
  "SerialNumber": "CN7016xxxxxxx",
  "Boot": {
    "BootSourceOverrideEnabled": "Disabled",
    "BootSourceOverrideMode": "UEFI",
    "BootSourceOverrideTarget": "None",
    "BootSourceOverrideTarget@Redfish.AllowableValues":
      ["None", "Pxe", "Hdd", "Cd", "BiosSetup", "UefiTarget", "UefiHttp"]
  },
  "Actions": {
    "#ComputerSystem.Reset": {
      "target": "/redfish/v1/Systems/System.Embedded.1/Actions/ComputerSystem.Reset",
      "ResetType@Redfish.AllowableValues":
        ["On", "ForceOff", "GracefulShutdown", "GracefulRestart",
         "ForceRestart", "Nmi", "PowerCycle"]
    }
  }
}
```

**Set a one-time PXE boot** — `Once` means the BMC reverts
`BootSourceOverrideEnabled` to `Disabled` after the next boot, which is what you want:
if provisioning fails you do not get an infinite netboot loop, you get a machine that
boots its disk and can be logged into.

```
PATCH /redfish/v1/Systems/System.Embedded.1
Content-Type: application/json
```
```json
{ "Boot": { "BootSourceOverrideEnabled": "Once",
            "BootSourceOverrideMode":    "UEFI",
            "BootSourceOverrideTarget":  "Pxe" } }
```

**Power the machine:**

```
POST /redfish/v1/Systems/System.Embedded.1/Actions/ComputerSystem.Reset
```
```json
{ "ResetType": "ForceRestart" }
```

`GracefulRestart` asks the OS to shut down; on a machine with no OS it does nothing and you
wait out a timeout. `ForceRestart` and `ForceOff` are the honest choices during
provisioning. `PowerCycle` behaves like removing and restoring power, which is what you
want when the BMC and the host are disagreeing about state.

**Virtual media — the escape hatch from PXE entirely.** Redfish lets you attach a remote
ISO as a virtual CD-ROM over HTTP(S), then boot from it:

```
POST /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia
{ "Image": "http://10.0.0.5:7080/iso/b8-ce-f6-00-07-aa/hook.iso", "Inserted": true,
  "WriteProtected": true }

PATCH /redfish/v1/Systems/System.Embedded.1
{ "Boot": { "BootSourceOverrideEnabled": "Once", "BootSourceOverrideTarget": "Cd" } }
```

and `VirtualMedia.EjectMedia` afterwards. Metal3's
`redfish-virtualmedia://…` BMC address scheme selects exactly this path, and Tinkerbell's
`isoboot` mode drives the same three calls through a `bmc.tinkerbell.org/v1alpha1` `Job`.
The reason to care: **virtual media needs no DHCP, no TFTP and no layer-2 adjacency** —
just HTTPS from the BMC to your image server. In a network where you do not control DHCP
and cannot get a relay configured, it is often the only pipeline you can actually build.
The cost is that the ISO streams over the BMC's management NIC, which on many boards is a
1 GbE port shared with everything else the BMC does, so a 600 MB image takes minutes and
40 of them in parallel will not go well.

The MAC-address formatting in that ISO URL (`b8-ce-f6-00-07-aa`, dash-delimited) is not
cosmetic. Upstream's own docs note it as a workaround for BMC firmware that mishandles
colons in URLs; Smee supports colon, dash, dot and no-delimiter formats as a configuration
option for the same reason. When a URL 404s in a pipeline that worked last week on
different hardware, check the delimiter before you check anything else.

### 10. Two control models: state machine versus workflow

Both models pass through the same seven stages. They differ in what you author and
therefore in what you version, review and diff.

**Ironic authors state.** A node moves through a documented finite state machine, and you
drive it by requesting transitions; Ironic derives the steps. The provisioning states are
defined in `ironic/common/states.py` (verified at commit `d275931`):

```
                       ┌──────────┐
                       │  enroll  │  BMC creds recorded, nothing verified
                       └────┬─────┘
                    manage  │  (BMC login proved, power state read)
                       ┌────▼──────────┐
                       │  verifying    │──fail──▶ enroll
                       └────┬──────────┘
                       ┌────▼──────────┐         ┌────────────┐
                       │  manageable   │◀───────▶│ inspecting │──fail──▶ inspect failed
                       └────┬──────────┘         └────────────┘
                   provide  │  (runs automated CLEANING on the way)
                       ┌────▼──────────┐
                       │   cleaning    │──▶ clean wait ──▶ clean failed
                       └────┬──────────┘         (agent is doing the work)
                       ┌────▼──────────┐
                       │   available   │  free inventory: nothing is deployed
                       └────┬──────────┘
                     deploy │
                       ┌────▼──────────┐
                       │   deploying   │──▶ wait call-back ──▶ deploy failed
                       └────┬──────────┘      (agent streams the image)
                       ┌────▼──────────┐
                       │    active     │  OS installed and booted; this is "provisioned"
                       └────┬──────────┘
                   undeploy │
                       ┌────▼──────────┐
                       │   deleting    │──▶ cleaning ──▶ available   (the loop closes)
                       └───────────────┘

  Also present and worth knowing: adopting (take over an already-installed node without
  re-imaging it), rescue / rescuing / unrescuing (boot a rescue ramdisk on a deployed
  node), and servicing / service wait (apply steps to an ACTIVE node — the state you use
  for in-place firmware updates without a full re-provision).
```

Metal3's `BareMetalHost` presents a simplified projection of that machine —
`registering → inspecting → preparing → available → provisioning → provisioned →
deprovisioning → deleting`, with `ready` retained as a deprecated alias for `available`
(`baremetalhost_types.go`, commit `f08172e`).

The full BMH for one GPU node:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gpu-node-07-bmc
  namespace: metal3
type: Opaque
stringData:
  username: provisioner
  password: "…"
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: gpu-node-07
  namespace: metal3
  labels:
    sku: hgx-h100-8g
    batch: "2026-Q1"                      # feeds lesson 06's per-SKU failure tracking
spec:
  online: true
  bootMACAddress: "b8:ce:f6:00:07:aa"     # the NIC that PXEs — NOT the BMC's MAC
  bootMode: UEFI                          # enum: UEFI | UEFISecureBoot | legacy
  automatedCleaningMode: metadata         # enum: metadata | disabled  (see §7)
  bmc:
    address: redfish-virtualmedia://10.0.7.7/redfish/v1/Systems/System.Embedded.1
    credentialsName: gpu-node-07-bmc
    disableCertificateVerification: false
  rootDeviceHints:
    rotational: false
    minSizeGigabytes: 900                 # excludes the 480 GB SATA boot device
    vendor: "SAMSUNG"                     # substring match
  image:
    url: http://images.metal3.svc/ubuntu-24.04-gpu-base.qcow2
    checksum: http://images.metal3.svc/ubuntu-24.04-gpu-base.qcow2.sha256
    checksumType: sha256                  # enum: md5 | sha256 | sha512 | auto
    format: qcow2
  userData:
    name: gpu-node-07-userdata            # cloud-init: kubeadm join + node labels
    namespace: metal3
  networkData:
    name: gpu-node-07-networkdata         # static addressing for the data NICs
    namespace: metal3
```

**Tinkerbell authors sequence.** A `Workflow` binds a `Hardware` record to a `Template`
whose `Task`s contain ordered `Action` containers, each with its own image, env, timeout,
retries and `background` flag (see §6). `bootOptions.bootMode` selects how the machine is
got into the agent in the first place — `netboot` (Tink Controller creates a BMC `Job`
that powers off, sets one-time PXE, powers on), `isoboot` (mount HookOS as virtual media,
eject it in the POST phase), or `customboot` (you write the BMC `Job` yourself).

**The one-line distinction, and why it is not just taste:** Ironic asks *"what state should
this host be in?"* and derives the steps; Tinkerbell asks *"what sequence of steps should
this host run?"* and executes them. The consequence is where your customisation lives. A
site-specific step in Ironic is a Python hardware manager shipped inside the IPA ramdisk
and a config change on the conductor; the same step in Tinkerbell is a container image and
three lines of YAML. Ironic gives you a richer state model for free (rescue, adopt,
service) that you would have to build yourself in Tinkerbell; Tinkerbell gives you a lower
barrier to arbitrary steps. Pick by which kind of change you expect to make often.

### 11. Idempotence, and what actually makes re-imaging safe

"Idempotent" gets asserted a lot. Four mechanisms make it real, and each fails in a
specific way if you skip it:

- **Declarative desired state.** The BMH or Workflow is the source of truth; the controller
  compares desired against observed and acts only on drift. Re-applying an unchanged
  manifest is a no-op. *Skip it* — drive provisioning from a script — and re-running the
  script re-images a live node.
- **Content-addressed images plus a checksum.** `image.url` with `checksum` and
  `checksumType` means "this exact image or fail." *Skip it* and a truncated download or a
  silently-replaced image installs successfully, and you find out weeks later when one
  node in forty behaves differently.
- **Cleaning between deprovision and provision.** Guarantees no stale partition table,
  filesystem superblock or RAID metadata survives. *Skip it* and the installer may find and
  reuse an old partition layout — the node comes up, joins, and has a root filesystem
  sized for the previous tenant.
- **Idempotent first-boot configuration.** cloud-init and Ignition are declarative and run
  once against a fresh disk; Talos's machine config is fully immutable by construction.
  *Skip it* — bootstrap with an imperative post-install script — and the node's final state
  depends on how many times the script ran.

The test that proves all four at once: pick a node, delete the Kubernetes `Node` object,
return the BMH to `available`, re-apply the identical manifest, and diff the resulting node
against a sibling. Same name, same labels, same kernel command line, same disk layout, same
join. If a human has to touch anything, one of the four is missing.

### 12. Provisioning throughput: how long does a rack actually take?

This is the arithmetic nobody does until the delivery date slips. Model each node as a
sequence of stages, mark which are per-node-independent and which contend for a shared
resource, and the makespan falls out.

**Per-node stage durations** (representative for an 8-GPU x86 server; measure your own —
these are the shapes, not your numbers):

| Stage | Duration | Contends for |
|---|---|---|
| BMC power-on → firmware POST complete | 4–8 min | nothing (a big server POSTs slowly; memory training on 2 TB of DDR5 dominates) |
| DHCP + iPXE binary (TFTP, ~1 MB) | 5–20 s | DHCP server, TFTP server |
| Kernel + initrd (~600 MB, HTTP) | `600 MB / per-node share of image-server BW` | **image server bandwidth** |
| Agent boot + inventory | 60–90 s | control plane API |
| Metadata clean | 10–30 s | nothing |
| Firmware/BIOS apply (when a change is pending) | 5–15 min + a reboot | nothing |
| Image write (~4 GB compressed → disk) | `4 GB / per-node share` + NVMe write time | **image server bandwidth** |
| Reboot into installed OS + cloud-init + join | 4–7 min | control plane API |

**The bottleneck is the image server's NIC, essentially always.** Do the arithmetic for
`N = 40` nodes:

```
  Bytes each node must pull:
      initrd + kernel                    0.6 GB
      OS image (compressed)              4.0 GB
      ────────────────────────────────────────
      per node                           4.6 GB
      × 40 nodes                       184.0 GB   of egress from the image server

  Image server on a single 10 GbE link, ~85% of line rate achievable in practice:
      10 Gb/s × 0.85 / 8               = 1.06 GB/s
      184 GB / 1.06 GB/s               = 174 s of pure transfer  ... per WAVE

  But transfer is only part of a node's path. Total per-node serial time
  (POST + agent + clean + image + reboot + join), with no contention:
      6 min POST + 1.5 min agent + 0.5 clean + 4.3 min transfer
      + 5 min reboot/join              ≈ 17.3 min

  If all 40 start together, the 184 GB of transfer serialises on the link:
      makespan ≈ non-transfer time + total transfer time
              ≈ 13 min + 174 s        ≈ 15.9 min   ← link is NOT the constraint at 10 GbE
```

That last line is the surprise, and it is why you do the calculation instead of guessing:
at 4.6 GB per node, a 10 GbE image server serves a 40-node rack in under three minutes of
aggregate transfer. **The dominant term is POST and reboot — firmware, not network.** A
2 TB-DDR5 server spends five to eight minutes training memory before it emits a single
DHCP packet, and it does that twice (once to reach the agent, once to boot the installed
OS).

The picture inverts as soon as any of three things change:

```
  Sensitivity — what actually moves the makespan for N = 40

  (a) Uncompressed 40 GB image instead of 4 GB compressed
        1,624 GB / 1.06 GB/s = 1,532 s = 25.5 min of transfer
        makespan ≈ 13 + 25.5 = 38.5 min          ← now the link dominates. 2.4×.
        Fix: compress (image2disk's COMPRESSED=true), or serve over a 25/100 GbE link,
             or put a caching proxy in each rack.

  (b) Full-erase clean on 8 × 3.84 TB NVMe
        NVMe format/crypto-erase: seconds.  Block overwrite: 30.7 TB / 6 GB/s ≈ 85 min.
        makespan ≈ 13 + 3 + 85 = 101 min         ← 6× worse, and entirely a config choice.
        Fix: confirm the drives support and are configured for secure erase
             (ironic's [deploy] enable_nvme_secure_erase), do not silently fall back.

  (c) Virtual media instead of PXE, BMC on a shared 1 GbE port
        0.6 GB ISO at ~60 MB/s effective = 10 s ... per node, but the BMC link is
        per-node, so this parallelises. The trap is a shared management switch uplink:
        40 × 0.6 GB = 24 GB over a 1 GbE uplink = 200 s, plus the BMC's own slow I/O.
        Fix: stage ISOs on a per-rack HTTP server, not a central one.

  (d) Firmware update pending on every node
        +5–15 min and one extra reboot, unavoidable and worth it. Budget it explicitly
        for a first-touch rack; it should be zero on a re-image.
```

**The operational rule:** measure the four numbers that appear in these formulas for your
own hardware — POST time, image size on the wire, image-server egress bandwidth, and
whether your clean step is metadata or full erase — and you can predict a rack bring-up to
within a few minutes. Guess at them and you will be wrong by a factor of six in the
direction that ruins a maintenance window.

One more constraint that is not bandwidth: **concurrency limits in the control plane
itself.** Ironic's conductor has a bounded worker pool and per-node locks; running 200
simultaneous deployments against one conductor queues them regardless of how much
bandwidth you have. This is a real capacity-planning input at fleet scale and the reason
Ironic supports multiple conductors with a consistent hash ring distributing nodes across
them.

### 13. The structural picture: who talks to whom

```
                    ┌──────────────────────── KUBERNETES API ────────────────────────┐
                    │  Cluster · MachineDeployment · Machine        (lesson 04)      │
                    │  Metal3Machine ─┐                Hardware ─┐                   │
                    │  BareMetalHost ◀┘                Workflow ◀┘  Template         │
                    │  HostFirmwareSettings            bmc.tinkerbell.org: Machine,  │
                    │  HostFirmwareComponents                        Job, Task       │
                    └───────┬─────────────────────────────────────────┬──────────────┘
                            │ watch/reconcile                         │ watch/reconcile
                  ┌─────────▼──────────┐                    ┌─────────▼──────────┐
                  │ baremetal-operator │                    │  Tink Controller   │
                  │        +           │                    │        +           │
                  │   Ironic (API,     │                    │  Rufio (BMC ctrl)  │
                  │   conductor, TFTP, │                    │  Smee (DHCP/TFTP/  │
                  │   httpd, inspector)│                    │   HTTP/iPXE/syslog)│
                  └────┬──────────┬────┘                    └────┬──────────┬────┘
      Redfish/IPMI ────┘          └──── DHCP/TFTP/HTTP            │          │
      (mgmt VLAN)                       (provisioning VLAN)       │          │
             │                                    │              │          │
   ┌─────────▼─────────┐              ┌───────────▼──────────┐   │          │
   │  BMC  10.0.7.7    │              │  NIC  b8:ce:f6:…:aa  │   │          │
   │  ─ power on/off   │              │  ─ PXE ROM → iPXE    │◀──┘          │
   │  ─ boot override  │              │  ─ kernel + initrd   │              │
   │  ─ virtual media  │              └───────────┬──────────┘              │
   │  ─ sensors, SEL   │                          │                         │
   └───────────────────┘              ┌───────────▼──────────┐              │
                                      │  AGENT OS (in RAM)   │──────────────┘
                                      │  IPA  or  HookOS     │  gRPC 42113 / HTTPS
                                      │  ─ inventory         │  reports inventory,
                                      │  ─ clean             │  pulls next step
                                      │  ─ firmware apply    │
                                      │  ─ write image ──────┼──▶ ┌──────────────┐
                                      └───────────┬──────────┘    │ IMAGE SERVER │
                                                  │               │ qcow2 / raw  │
                                                  │ reboot        │ + .sha256    │
                                      ┌───────────▼──────────┐    └──────────────┘
                                      │  INSTALLED OS on disk│
                                      │  cloud-init/Ignition │──▶ kubeadm join / talosctl
                                      │  kubelet, containerd │    (control plane, L01/03)
                                      └───────────┬──────────┘
                                                  │  node registers
                                      ┌───────────▼──────────┐
                                      │  NFD labels from PCI │  feature.node.kubernetes.io/
                                      │  vendor 0x10de       │  pci-10de.present=true
                                      │  class  0x0302       │
                                      └───────────┬──────────┘
                                                  ▼
                                    ═══ HANDOFF LINE (lesson 04) ═══
                                     GPU Operator selects on the label,
                                     installs driver + toolkit + device
                                     plugin + DCGM. Node then advertises
                                     nvidia.com/gpu: 8.

  THREE NETWORKS, and conflating them is a classic bring-up bug:
    · management VLAN   — BMCs only. Redfish/IPMI. Never routable from tenants.
    · provisioning VLAN — DHCP/TFTP/HTTP + the host's PXE NIC. Needs an ip
                          helper-address relay if the provisioning service is not
                          layer-2 adjacent.
    · data/tenant VLANs — configured by networkData/cloud-init AFTER the OS lands.
                          They do not exist during stages ①–⑥.
```

### 14. Where this hands off, and why drivers stay out of the image

Provisioning's job ends at **"`Ready` Kubernetes node with NVIDIA hardware present and
labelled."** Concretely, stage ⑦ finishes when:

- kubelet and containerd are up and the node has joined the control plane built in lessons
  01 and 03 (via `kubeadm join` with a bootstrap token, or `talosctl apply-config`);
- **Node Feature Discovery** has labelled the node from its PCI inventory — NVIDIA's PCI
  vendor ID `0x10de` and class `0x0302` (3D controller) produce
  `feature.node.kubernetes.io/pci-10de.present=true`;
- the node is `Ready` and has **no GPU driver, no container toolkit, no device plugin**.

The GPU Operator (lesson 04) then selects on that label and does the driver rollout,
container-toolkit install, device-plugin registration and DCGM deployment.

Keeping the driver out of the golden image is a deliberate coupling decision, and the
argument is worth stating as a rate comparison: OS images change on the order of quarterly
(security baselines, kernel LTS bumps); CUDA and driver versions change on the order of
monthly, and sometimes per-tenant. Bake them together and the *faster* clock forces the
*slower* pipeline — every driver bump becomes a full re-image of every node, at the
makespan you computed in §12. Keep them separate and a driver bump is a DaemonSet rollout
measured in minutes with no reboot in the common case. **Couple two lifecycles and you
inherit the union of their change rates, at the cost of the more expensive one.**

## Perspectives

**The operator's view.** You watch a `BareMetalHost` or a Tinkerbell `Workflow` walk
through phases and your job is to make the manifest right once. The skill that matters is
reading a stuck object: which phase, how long, and which of the three networks that phase
depends on. `kubectl get bmh -A -o wide` plus the Ironic conductor log answers most of it;
the serial console answers the rest, which is why you wire serial before you need it.

**The hardware and firmware view.** Nothing above stage ⑤ exists without a correctly
configured BMC, and BMC firmware quality varies enormously between vendors and between
firmware revisions on the same board. Redfish is a standard with a large optional surface:
one board implements `UefiHttp` boot override, another does not; one accepts a colon in a
virtual-media URL, another 404s. Treat the BMC as a small, unreliable embedded web server
that occasionally needs its own reset, because that is what it is.

**The network engineer's view.** Two of the six failure modes in §4 are switch
configuration, and they present as software bugs. A port that has not finished
spanning-tree convergence when the PXE ROM sends its DISCOVER produces "no offer received"
with a perfectly healthy DHCP server; `portfast`/edge-port configuration on provisioning
ports is not an optimisation, it is a requirement. A provisioning service that is not
layer-2 adjacent needs `ip helper-address` on the SVI, and — if you are using ProxyDHCP —
that relay has to forward to *both* servers.

**The economics view.** Every hour a delivered rack is not provisioned is depreciation
with no offsetting utilisation: `$128/node-day` on the numbers at the top of this lesson,
before power or opportunity cost. The pipeline is also what makes lesson 06's remediation
loop cheap — a node you can re-image unattended in 17 minutes is a node you can
aggressively recycle on a soft fault instead of babysitting. And the per-node firmware and
image versions this pipeline records are the raw material for the per-SKU failure-rate
tracking in lesson 06 that turns "we have had some hardware problems" into a vendor-facing
claim.

## Real-world use cases

- **Tinkerbell in CNCF and the consolidation into one binary.** Tinkerbell's components —
  Smee (DHCP/TFTP/HTTP/iPXE/syslog), Tink Server and Controller (the workflow engine),
  Tootles (an EC2-compatible metadata service, **renamed from Hegel**), Rufio (BMC control
  via `bmc.tinkerbell.org` `Machine`/`Job`/`Task` CRDs) and SecondStar (SSH-to-serial over
  the BMC) — now ship as a **single `tinkerbell` binary** with per-service enable flags,
  listening on 7080/TCP (HTTP), 7443/TCP (HTTPS), 42113/TCP (gRPC), 67/UDP (DHCP), 69/UDP
  (TFTP), 514/UDP (syslog) and 2222/TCP (SSH). **What it shows:** the multi-service
  deployment diagrams in older write-ups are out of date, and if you are reading a blog post
  that talks about "Boots" and "Hegel" as separate deployments, it predates this
  consolidation. Verified directly against the repository (`tinkerbell/tinkerbell`, commit
  `725c33d`, `docs/technical/PORTS_AND_ENDPOINTS.md`, read 2026-08-18).
- **Metal3 as the Kubernetes-native front end to a fifteen-year-old state machine.** The
  `BareMetalHost` CRD is a deliberately narrow projection of Ironic's full node state
  machine — Metal3 exposes eight provisioning states where Ironic has more than thirty,
  and exposes `automatedCleaningMode` as a two-valued enum over a priority-based clean-step
  system with many more settings. **What it shows:** when a BMH sticks in a state with an
  unhelpful message, the answer is usually in the Ironic layer underneath, not in the CRD.
  Knowing that Metal3 is a projection — and where to look under it — is the difference
  between a 10-minute and a 3-hour debug. Verified against `metal3-io/baremetal-operator`
  (`f08172e`) and `openstack/ironic` (`d275931`), read 2026-08-18.
- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for ML Training
  and Inference?"** ([coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference](https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference))
  — describes periodic **HPC Verification** tests, roughly 20 minutes long, exercising all
  GPUs on a node, as part of the provisioning-to-health pipeline. **What it shows:** stage
  ⑦ is not the last gate. A burn-in that exercises every GPU, every NVLink and every rail
  belongs between "node is `Ready`" and "node is trusted with a customer's job", and the
  20-minute figure is a useful anchor for what "long enough to be meaningful" costs. This
  is the same post lesson 06 anchors on, which is the point: provisioning and remediation
  are two arcs of one loop. *(Page not fetched through this session's egress proxy; cited
  from the module's existing research notes.)*

## Worked example: `gpu-node-07`, from rack to handoff

An HGX H100 node has been racked, cabled to the management VLAN and the provisioning VLAN,
and powered. Nothing else has been done.

**Step 0 — record the facts you have.** The technician's ticket gives you the BMC IP
(`10.0.7.7`), BMC credentials, the chassis serial, and — critically — the MAC of the NIC
patched to the provisioning VLAN (`b8:ce:f6:00:07:aa`). That MAC is the machine's only
identity until the agent runs. Getting it wrong is the second-most-common bring-up error
after option 77; if the node has four NICs and you record the wrong one, the machine boots
correctly and never talks to you.

**Step 1 — prove the BMC before you automate anything.**

```
$ curl -sk -u provisioner:'…' https://10.0.7.7/redfish/v1/Systems/System.Embedded.1 \
    | jq '{PowerState, SerialNumber, Boot: .Boot.BootSourceOverrideTarget,
           Allowed: .Boot."BootSourceOverrideTarget@Redfish.AllowableValues"}'
{
  "PowerState": "Off",
  "SerialNumber": "CN7016xxxxxxx",
  "Boot": "None",
  "Allowed": ["None","Pxe","Hdd","Cd","BiosSetup","UefiTarget","UefiHttp"]
}
```

Three facts confirmed in one call: credentials work, the machine is off, and this BMC
supports both `Pxe` and `Cd` (virtual media) overrides — so both pipelines are available
to you. If `Pxe` were absent from the allowable values you would know immediately to build
the virtual-media path instead of debugging DHCP for two days.

**Step 2 — enrol.** Apply the `Secret` and `BareMetalHost` from §10.
`baremetal-operator` registers the node in Ironic. Watch the projection:

```
$ kubectl -n metal3 get bmh gpu-node-07 -w
NAME          STATE          CONSUMER   ONLINE   ERROR   AGE
gpu-node-07   registering               true             8s
gpu-node-07   inspecting                true             41s
gpu-node-07   preparing                 true             6m12s
gpu-node-07   available                 true             9m03s
```

`registering → inspecting` took 33 s (BMC login, power-on command). `inspecting` took
5.5 minutes — nearly all of it firmware POST and memory training, exactly as §12 predicts.

**Step 3 — read the inventory as an acceptance test.** This is the step people skip.

```
$ kubectl -n metal3 get bmh gpu-node-07 -o jsonpath='{.status.hardware}' | jq '{
    cpu: .cpu.count, ram: .ramMebibytes,
    nics: [.nics[] | {name, mac, speedGbps, pxe}],
    disks: [.storage[] | {name, sizeBytes, rotational, model}] }'
{
  "cpu": 224,
  "ram": 2064384,
  "nics": [
    {"name":"eno1","mac":"b8:ce:f6:00:07:aa","speedGbps":25,"pxe":true},
    {"name":"ens6f0","mac":"b8:ce:f6:00:07:b0","speedGbps":400,"pxe":false},
    …
  ],
  "disks": [
    {"name":"/dev/nvme0n1","sizeBytes":1920383410176,"rotational":false,
     "model":"SAMSUNG MZQL21T9HCJR-00A07"},
    …
  ]
}
```

Now diff against the expected SKU profile. The one that matters for a GPU node is not in
this output: Ironic's default inspection reports CPU, memory, NICs and disks but not a
per-device PCI list unless you enable the PCI-devices inspection hook. **Turn it on and
assert the GPU count in the pipeline**, because the alternative is discovering at
scheduling time that this node has seven GPUs. Expected: eight devices with vendor `10de`
and class `0302`.

**Step 4 — firmware before OS.** With the BMH `available`, apply the
`HostFirmwareSettings` from §8. The operator diffs against what Redfish reports and
schedules the changes; the node cycles back through `preparing`, applies BIOS settings and
any pending component firmware, and reboots. Six minutes and one extra POST. On a
re-image of a node whose firmware is already at the declared baseline, this stage is a
no-op — which is the whole point of declaring it rather than scripting it.

**Step 5 — provision.** Setting `spec.image` (or having a `Metal3Machine` claim the host)
moves it to `provisioning`. Ironic tells the agent to stream the image, verifies the
checksum, writes `userData` and `networkData` to the config drive, sets next boot to disk,
and reboots. Watch the transfer rate, because it is the number from §12 you most need for
your own hardware:

```
$ kubectl -n metal3 logs deploy/ironic -c ironic-conductor | grep -i 'gpu-node-07'
… Node <uuid> deploy step {'step': 'write_image', 'priority': 80, …} started
… Image download and write completed in 231.4 seconds (4.10 GB, 17.7 MB/s effective)
… Node <uuid> deploy step {'step': 'prepare_instance_boot', …} started
… Node <uuid> moved to provision state "active" from state "deploying"
```

17.7 MB/s effective for a single node — well under the link's capacity, because this
transfer is decompress-and-write-bound at the agent, not network-bound. That is the number
that changes the §12 sensitivity analysis: if the *agent* is the bottleneck rather than the
link, provisioning 40 nodes in parallel is nearly free, and the makespan is dominated by
POST time as computed.

**Step 6 — join and label.**

```
$ kubectl get node gpu-node-07 -o wide
NAME          STATUS   ROLES    AGE   VERSION   INTERNAL-IP   OS-IMAGE
gpu-node-07   Ready    <none>   64s   v1.34.1   10.0.9.107    Ubuntu 24.04.3 LTS

$ kubectl get node gpu-node-07 -o json | jq '.metadata.labels
    | with_entries(select(.key|test("pci-10de|nvidia|sku")))'
{
  "feature.node.kubernetes.io/pci-10de.present": "true",
  "sku": "hgx-h100-8g"
}

$ kubectl get node gpu-node-07 -o jsonpath='{.status.allocatable}' | jq '."nvidia.com/gpu"'
null                       ← EXPECTED. No driver, no device plugin, no advertised GPUs.
```

That `null` is the handoff line. The node is `Ready`, it is labelled with what the hardware
is, and it advertises zero GPUs because nothing has installed a device plugin yet. The GPU
Operator's node selector matches `pci-10de.present=true` and takes it from here.

**Step 7 — prove idempotence.** Cordon and delete the `Node`, set the BMH back to
`available`, re-apply the identical manifests. Fifteen minutes later, diff:

```
$ diff <(kubectl get node gpu-node-07 -o json | jq -S '.metadata.labels, .status.nodeInfo') \
       ./golden/gpu-node-07.json
$ echo $?
0
```

Zero differences, no human input. That is the acceptance criterion for this lesson's
practice.

## Practice (feeds the deliverable)

**Design and demonstrate a netboot-to-Ready workflow for a GPU node.** Deliver into
[`practice/capex-vs-cloud/`](../practice/capex-vs-cloud/README.md):

1. **A message-sequence diagram** of the boot chain for one node, in the shape of §4:
   every hop, the protocol and port, the DHCP options carried, and — for each hop — the
   symptom you would observe if it failed. Label which of the three networks each hop uses.
2. **A config artifact** — a real `BareMetalHost` + `HostFirmwareSettings` (Metal3) *or* a
   real `Hardware` + `Template` + `Workflow` (Tinkerbell). It must include: image URL with
   checksum and checksum type, root device hints that do not name a device path, the
   firmware assertions that matter for an 8-GPU node (above-4G decoding, SR-IOV, IOMMU,
   UEFI, serial console), the cleaning mode with a one-line note on what it does and does
   not erase, and the cloud-init/Ignition that performs the cluster join.
3. **The provisioning-throughput calculation** for your fleet size, in the form of §12:
   state your per-node POST time, image size on the wire, image-server egress bandwidth
   and clean mode; compute the makespan for `N` nodes; then run the three sensitivities
   (uncompressed image, full-erase clean, virtual media over a shared management uplink)
   and name which one you are actually exposed to.
4. **An idempotency note** — the four mechanisms from §11 and, for each, the specific
   failure you would see without it.

**Hardware-free path.** All of this is demonstrable on VMs. Either (a) stand up a
Tinkerbell stack against libvirt VMs with `sushy-emulator` providing a Redfish endpoint —
this is the same `openstack/sushy` library Ironic drives, so the Redfish calls in §9 work
verbatim; or (b) generate a Talos Image Factory schematic that includes the NVIDIA
extension, boot a VM from the factory's iPXE URL, and `talosctl` it to `Ready`. Either way,
**capture the DHCP exchange** — `tcpdump -i br0 -vv 'port 67 or port 68'` — and identify
options 60, 77 and 93 in the packets. That capture is the evidence that you understand §2
rather than having copied a manifest.

**Acceptance:** the diagram, the config artifact, the throughput calculation with its
sensitivities, and the idempotency note, all committed, with the GPU-Operator handoff
boundary called out explicitly and a packet capture showing option 77 doing its job.

## Common pitfalls

- **Not matching on option 77, and getting an infinite iPXE chainload loop.** *Symptom:*
  the iPXE banner scrolls past every few seconds forever; the DHCP server log shows the
  same MAC requesting over and over. *Mechanism:* iPXE re-runs DHCP after it loads, and
  unless the server distinguishes "already iPXE" (user class `iPXE`, or your patched
  build's custom string) from "firmware ROM," it serves the bootloader again. This is the
  single most common bare-metal provisioning failure and it has exactly one fix.
- **Recording the BMC's MAC as `bootMACAddress`.** *Symptom:* the BMH sits in `inspecting`
  until it times out; the machine boots fine and reaches the OS installer prompt on its own.
  *Mechanism:* the provisioning service is waiting for a DHCP request from a MAC that will
  never send one, while the host's actual PXE NIC gets no boot instructions.
- **Reading `automatedCleaningMode: metadata` as "erases the disks."** *Symptom:* a
  compliance audit, or a previous tenant's data recoverable from a "wiped" node.
  *Mechanism:* metadata cleaning removes partition tables, superblocks and RAID metadata —
  pointers, not data. Full erasure is a separate Ironic clean step
  (`erase_devices`, default priority 10) that Metal3 does not expose as a BMH field.
- **Applying firmware after the OS is installed.** *Symptom:* a reboot in a maintenance
  window for every firmware drift, and BIOS settings that "did not take" because the
  setting requires a reset the tool did not perform. *Mechanism:* BIOS settings and PCIe
  enumeration are read by the kernel at boot; changing them under a running OS requires the
  reboot you were trying to avoid. Do it in the ramdisk where a reboot is free.
- **Trusting `Ready` without asserting the GPU count.** *Symptom:* a training job schedules
  onto a node and NCCL reports a topology mismatch, weeks later. *Mechanism:* `Ready`
  means the kubelet is happy. A node with above-4G decoding disabled can enumerate a subset
  of GPUs, pass every Kubernetes health check, and be wrong. Diff inspection output against
  the expected device count in the pipeline.
- **Copying a `HostFirmwareSettings` manifest between vendors.** *Symptom:* the setting
  silently does not apply and nothing errors. *Mechanism:* BIOS attribute names are
  vendor-specific and an unknown key is not an error in most implementations. Read the
  `FirmwareSchema` object for the host in front of you.
- **No checksum on the image.** *Symptom:* one node in forty behaves differently and no
  one can explain it. *Mechanism:* without `checksum`/`checksumType`, a truncated or
  substituted image installs successfully. The entire idempotence argument depends on
  "this exact image or fail."
- **Treating IPMI and Redfish as interchangeable.** *Symptom:* a pipeline that works on one
  vendor's boards and fails on another's. *Mechanism:* IPMI has no standard virtual-media
  story, a weak and OEM-extended data model, and no schema. Redfish's data model is
  standard but large parts are optional — always read
  `BootSourceOverrideTarget@Redfish.AllowableValues` from the machine rather than assuming.

## Self-check

**(a) A rack of new nodes all sit at the iPXE banner, cycling every few seconds. What is
almost certainly wrong, and what is the mechanism?**
**Answer:** The DHCP/boot server is not distinguishing an iPXE client from a firmware PXE
ROM, so it keeps answering with the iPXE binary in option 67 and iPXE keeps chainloading
itself. The distinguishing signal is **DHCP option 77 (user class)**, which iPXE sets
unconditionally to the string `iPXE` in every request — it is hard-coded in iPXE's
`dhcp_request_options_data` table, not configurable. The server's rule must be: option 77
equals `iPXE` (or your custom patched string, e.g. `Tinkerbell`) → serve the **script**;
otherwise → serve the **binary**. Confirm with `tcpdump -i <if> -vv 'port 67 or port 68'`
and look for option 77 in the second request. Note the second-order case: if you deploy a
vendor-patched iPXE that advertises a *different* user class, you must match that string
too or you loop one level deeper.

**(b) Where does firmware update slot into the boot sequence, and why before the OS image?**
**Answer:** Inside the ephemeral agent ramdisk (IPA or HookOS), after inspection and
cleaning and **before** the OS is written. Four independent reasons: (1) the kernel
enumerates PCIe based on BIOS state, so above-4G decoding, resizable BAR, SR-IOV and IOMMU
settings must be correct before the OS ever boots, or the eight GPU BARs cannot be mapped
above the 4 GB line; (2) firmware flashes require a reboot you do not want to inflict on a
production OS; (3) cleaning is destructive and cannot run under the OS it is about to
overwrite; (4) determinism — every node reaching an identical firmware baseline before
imaging leaves the image as the only variable, which is what makes "re-image and see if it
recurs" a valid diagnostic rather than a coin flip.

**(c) How do you make a node re-image idempotent, and what breaks for each mechanism you
omit?**
**Answer:** Four mechanisms. **Declarative desired state** (the BMH or Workflow is the
truth; the controller acts only on drift) — omit it and re-running your provisioning script
re-images a live node. **Content-addressed image plus checksum** (`checksum`,
`checksumType`) — omit it and a truncated or substituted image installs silently, producing
one node in forty that behaves differently. **Cleaning between deprovision and provision**
— omit it and the installer can find and reuse a stale partition table, giving you a root
filesystem sized for the previous tenant. **Idempotent first-boot config** (cloud-init,
Ignition, or Talos's immutable machine config) — omit it and the node's final state depends
on how many times a post-install script ran. The test that proves all four: delete the
`Node`, return the BMH to `available`, re-apply the same manifests, and diff the result
against a golden copy — zero differences, no human input.

**(d) Where exactly does bare-metal provisioning hand off to lesson 04's GPU Operator, and
why are drivers deliberately not in the image?**
**Answer:** At a `Ready` node with NVIDIA hardware **present and labelled but with no
driver**: kubelet and containerd up, `kubeadm`/Talos joined, NFD having labelled the node
from PCI inventory (vendor `0x10de`, class `0x0302` →
`feature.node.kubernetes.io/pci-10de.present=true`), and
`status.allocatable["nvidia.com/gpu"]` still absent. The GPU Operator selects on that label
and installs driver, container toolkit, device plugin and DCGM. Drivers stay out of the
image because OS images change quarterly while drivers change monthly: coupling the two
lifecycles means the faster clock drives the slower, more expensive pipeline, so every CUDA
bump becomes a full fleet re-image at the makespan computed in §12 instead of a DaemonSet
rollout measured in minutes.

**(e) Estimate the makespan to provision 40 nodes, and name what you would check first if
it came in three times longer.**
**Answer:** Model it as per-node serial time plus contention on the shared resource. With
6 min POST, 1.5 min agent boot and inventory, 0.5 min metadata clean, ~4.3 min image
transfer and ~5 min reboot-plus-join, one node is ~17 min. The shared resource is image
server egress: 40 nodes × 4.6 GB (initrd + compressed image) = 184 GB, and at 1.06 GB/s
(10 GbE at ~85% of line rate) that is 174 s of aggregate transfer — so the link is *not*
the constraint and the makespan is roughly 16 min, dominated by firmware POST and the
second reboot. If it came in three times longer, check in this order: **(1)** is the image
being served uncompressed? 40 GB instead of 4 GB puts 25 min of transfer on the link and
makes it the dominant term; **(2)** is a full `erase_devices` clean running instead of
metadata-only, and is NVMe secure erase actually being used rather than a block overwrite
(30 TB of overwrite is ~85 min per node); **(3)** is a firmware update pending on every
node, adding 5–15 min and a reboot; **(4)** is the control plane itself the limit — one
Ironic conductor with a bounded worker pool queues deployments regardless of bandwidth.

**(f) A BMC does not list `Pxe` in `BootSourceOverrideTarget@Redfish.AllowableValues`. What
is your pipeline?**
**Answer:** Virtual media. `POST` to
`/redfish/v1/Managers/{id}/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia` with the agent
ISO's HTTP URL, `PATCH` the system's `Boot` object with
`{"BootSourceOverrideEnabled":"Once","BootSourceOverrideTarget":"Cd"}`, `POST`
`ComputerSystem.Reset` with `{"ResetType":"ForceRestart"}`, and `EjectMedia` afterwards.
This is what Metal3's `redfish-virtualmedia://` address scheme and Tinkerbell's `isoboot`
mode drive. The advantage is that it needs **no DHCP, no TFTP and no layer-2 adjacency** —
only HTTPS from the BMC to your image server — which is often decisive when you do not
control DHCP on that VLAN. The costs: the ISO streams over the BMC's management NIC, which
is frequently a shared 1 GbE port, so per-node transfer is slow and a central image server
behind a 1 GbE management uplink will not serve 40 nodes at once; stage ISOs per rack. Also
watch the MAC delimiter in the media URL — many BMCs mishandle colons, which is why
upstream uses dash-delimited MACs in ISO paths.

## Connections & what's next

This lesson is the physical floor under lesson 04's object model: a `Machine` is only as
real as the `BareMetalHost` or `Workflow` reconciling beneath it, and every CAPI operation
from lesson 04 — scaling a `MachineDeployment`, a rolling upgrade, applying a new Talos
machine config — ultimately triggers the seven stages here on real hardware. It feeds
module 09's networking concerns directly (the NIC firmware version you pinned in stage 5 is
what decides whether the rail negotiates 400G, and therefore what NCCL gets), and it feeds
lesson 08's economics (a slow or manual pipeline is depreciation with no utilisation
against it).

The thread that carries forward: this lesson ends at a labelled, driverless `Ready` node —
and that is also where the *back edge* of lesson 06's loop lands. When a node fails hard
enough to be RMA'd, the replacement does not get installed by hand; it re-enters this exact
pipeline (`BareMetalHost` back to `available` → clean → re-image → rejoin), at the makespan
you computed in §12. Lesson 06 is what decides *when* a node comes back through here: the
detect → isolate → decide → RMA loop that closes around the pipeline you just built. The
inspection output from §7 is the same data that lesson 06's per-SKU failure tracking keys
off, and the 17-minute re-provision time is what makes "just re-image it" a legitimate
first response to a soft fault instead of a gamble.

## References & further reading

**Primary sources (verified against upstream source this session)**

- **iPXE** — <https://ipxe.org/> — *`ipxe.org` is blocked by this session's egress proxy and
  was not read.* All iPXE facts above are verified against the upstream repository at
  `github.com/ipxe/ipxe`, commit `e6d0a97`: DHCP option codes and the iPXE encapsulated
  option space in `src/include/ipxe/dhcp.h` (option 175 with sub-options priority `0x01`,
  `no-pxedhcp` `0xb0`, scriptlet `0x51`); the architecture enum and its provenance note
  (PXE spec → RFC 4578 → RFC 5970 → IANA registry); the `PXEClient:Arch:%05u:UNDI:%03u%03u`
  vendor-class format; and the hard-coded `user-class = "iPXE"` in
  `src/net/udp/dhcp.c`'s `dhcp_request_options_data`.
- **Tinkerbell** — <https://tinkerbell.org/> — *not fetched; blocked.* Verified against
  `github.com/tinkerbell/tinkerbell`, commit `725c33d`: the single-binary consolidation and
  full port/endpoint table (`docs/technical/PORTS_AND_ENDPOINTS.md`); the four DHCP modes
  (`docs/technical/DHCP_BOOT_MODES.md`); the `netboot`/`isoboot`/`customboot` workflow boot
  modes and the exact `bmc.tinkerbell.org/v1alpha1` `Job` bodies they generate
  (`docs/technical/BOOT_MODES.md`); the architecture→binary map and the option-77 switch in
  `smee/internal/dhcp/dhcp.go`; the HookOS iPXE script template in
  `smee/internal/ipxe/script/hook.go`; and the `Action` schema including `background`,
  `retries` and `timeoutSeconds` in `api/v1alpha2/tinkerbell/task.go`. **Correction applied
  from this source:** the metadata service is now **Tootles**, not Hegel, and the components
  ship as one binary rather than separate deployments.
- **Metal3 baremetal-operator** — <https://github.com/metal3-io/baremetal-operator> —
  commit `f08172e`, `apis/metal3.io/v1alpha1/baremetalhost_types.go`. Source for the
  provisioning-state list, the `bootMode` enum (`UEFI`/`UEFISecureBoot`/`legacy`), the
  `checksumType` enum, the `Image` fields (including OCI-registry support via
  `ociAuthSecretName`), and the `RootDeviceHints` per-field matching semantics (substring
  for `model`/`vendor`, exact for the rest). **Correction applied:** `automatedCleaningMode`
  is a two-valued enum, `metadata` or `disabled` — it does not offer a full-erase mode, and
  `metadata` erases partition/filesystem/RAID metadata only.
- **OpenStack Ironic** — <https://github.com/openstack/ironic> — commit `d275931`.
  Source for the full provisioning state machine (`ironic/common/states.py`) and for the
  cleaning-step priorities in `ironic/conf/deploy.py` and
  `ironic/drivers/modules/agent_base.py`: `erase_devices` defaults to priority 10 and
  `erase_devices_metadata` to 99 in the `GenericHardwareManager`, with NVMe secure erase
  via `nvme-cli format` available in user-data and crypto modes.
- **OpenStack Sushy** — <https://github.com/openstack/sushy> — commit `3e7bd47`. Source for
  the Redfish enums used in §9: `BootSource` (`Pxe`, `Hdd`, `Cd`, `BiosSetup`, `UsbCd`, …),
  `BootSourceOverrideMode` (`Legacy`, `UEFI`), `BootSourceOverrideEnabled` (`Disabled`,
  `Once`, `Continuous`), `ResetType` (`On`, `ForceOff`, `GracefulShutdown`,
  `GracefulRestart`, `ForceRestart`, `Nmi`, `ForceOn`, `PushPowerButton`, `PowerCycle`), and
  the `VirtualMedia.InsertMedia` / `VirtualMedia.EjectMedia` actions.
- **IANA processor-architecture registry (DHCP option 93 values)** — *`iana.org` is blocked;
  not fetched.* Values verified against the registry mirror in
  `github.com/insomniacslk/dhcp`, `iana/archtype.go` (`master`, read 2026-08-18), which
  Tinkerbell's DHCP handler consumes: 0 Intel x86PC, 6 EFI IA32, 7 EFI x86-64, 9 EFI BC,
  10/11 EFI ARM32/ARM64, 15/16 EFI x86 / x86-64 boot from HTTP, 19 EFI ARM64 boot from HTTP.
- **DMTF Redfish specification and schema bundle** — <https://www.dmtf.org/standards/redfish>
  — *blocked by this session's egress proxy; not fetched and not relied upon.* Listed as the
  normative reference for the resource model; every Redfish detail above is instead sourced
  from Sushy and Ironic as noted.
- **RFC 4578 (PXE client options 93/94/97), RFC 5970 (UEFI DHCPv6 boot), RFC 2348 (TFTP
  block-size option), RFC 7440 (TFTP windowsize)** — *`rfc-editor.org` and
  `datatracker.ietf.org` are blocked; not fetched and not relied upon.* Listed for depth;
  the option semantics used above come from iPXE's and Tinkerbell's implementations.
- **UEFI Specification, HTTP Boot chapter** — <https://uefi.org/specifications> —
  *blocked; not fetched and not relied upon.* Listed as the normative source for HTTP Boot;
  the `HTTPClient` vendor-class behaviour described above comes from Tinkerbell's
  `IsNetbootClient` and `Bootfile` implementations.

**Real-world engineering**

- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for ML Training
  and Inference?"** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference>
  — **what it shows:** the ~20-minute HPC Verification burn-in that belongs between "node
  is `Ready`" and "node is trusted with a job"; the shared anchor with lesson 06.
  *(Not fetched this session — the domain is blocked by the egress proxy. Cited from the
  module's existing research notes rather than a fresh read.)*
- **Sidero Labs / Equinix case study — Kubespray to Talos** —
  <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security>
  — **what it shows:** cluster deployment time falling from ~45 minutes to under 10 by
  removing imperative steps between "machine exists" and "cluster member" — evidence that
  pipeline *design*, not hardware speed, moves this number. *(Not fetched this session;
  domain blocked. Cited from the module's existing research notes.)*

**Deeper dives**

- **Cluster API book, bare-metal provider chapters** — <https://cluster-api.sigs.k8s.io/> —
  for tracing how a `Machine` from lesson 04 claims and drives a `BareMetalHost` through
  CAPM3, or a `Hardware`/`Workflow` through CAPT.
- **Talos Image Factory** — <https://factory.talos.dev/> — schematic-based, extension-
  included (NVIDIA) iPXE images; the fastest hardware-free way to exercise a real netboot
  chain end to end.
