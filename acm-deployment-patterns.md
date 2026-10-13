# ACM Deployment Patterns

A summary of Red Hat Advanced Cluster Management (ACM) hub-of-hubs deployment patterns.

## References

| Resource | URL |
|---|---|
| AutoShift v2 repo | https://github.com/auto-shift/autoshiftv2 |
| AutoShift v2 docs site | https://auto-shift.github.io/autoshiftv2/ |
| Hub-of-hubs topology | https://auto-shift.github.io/autoshiftv2/hub-of-hubs/ |
| Cluster provisioning (BM, AWS, vSphere) | https://auto-shift.github.io/autoshiftv2/cluster-install/ |
| Quick start | https://auto-shift.github.io/autoshiftv2/quickstart/ |
| Values reference (all labels) | https://auto-shift.github.io/autoshiftv2/values-reference/ |

---

## Pattern 1: Standard Greenfield Hub-of-Hubs

The canonical starting point. All ACM tiers are separate, independent OpenShift clusters.

### Deployment sequence

```
Step 1: Bootstrap Cluster 0

  ┌──────────────────────────────────┐
  │  Cluster 0  (bare metal or any)  │
  │  - ACM Global Hub (HoH)          │
  │  - OpenShift GitOps (Argo CD)    │
  │  - AutoShift v2                  │
  └──────────────────────────────────┘


Step 2: From Cluster 0, provision Clusters 1..N

  ┌──────────────────────────────────┐
  │  Cluster 0  (ACM Global Hub)     │
  │                                  │
  │  provisions / imports ──────────►│  Cluster 1  (OCP)
  │                           ──────►│  Cluster 2  (OCP)
  │                           ──────►│  Cluster N  (OCP)
  └──────────────────────────────────┘


Step 3: Configure Clusters 1..N as ACM Hubs (via AutoShift policy)

  ┌──────────────────────────────────┐
  │  Cluster 0  (ACM Global Hub)     │
  │  AutoShift policies install:     │
  │  - ACM on Cluster 1              │
  │  - ACM on Cluster 2              │
  │  - ACM on Cluster N              │
  └──────────────────────────────────┘


Step 4: From each ACM Hub 1..N, create and import managed spoke clusters

  ACM Hub 1  ──► Spoke A, Spoke B, Spoke C
  ACM Hub 2  ──► Spoke D, Spoke E
  ACM Hub N  ──► Spoke ...
```

### Full topology

```
                    ┌────────────────────────────────────┐
                    │   Cluster 0 — ACM Global Hub (HoH) │
                    │   AutoShift v2 (policies-autoshift) │
                    └──────┬──────────────┬──────────────┘
                           │              │
               ┌───────────▼──┐      ┌───▼──────────────┐
               │  ACM Hub 1   │      │    ACM Hub N      │
               │  (Cluster 1) │      │    (Cluster N)    │
               │  AutoShift   │      │    AutoShift       │
               │  (policies-  │      │    (policies-      │
               │   hub1)      │      │     hubN)          │
               └──┬───────────┘      └──────┬────────────┘
                  │                         │
       ┌──────────┼──────────┐         ┌────▼────────────┐
       │          │          │         │   Spoke Cluster  │
       ▼          ▼          ▼         └─────────────────┘
   Spoke A    Spoke B    Spoke C
```

### Key rules (from ACM architecture)

- A cluster is managed by **exactly one** ACM instance.
- A hub cannot configure itself — it is configured from the tier above.
- The Global Hub cannot see leaf spokes (two boundaries away).
- AutoShift v2 implements this pattern end-to-end through Helm charts and ACM governance policies.

---

## Pattern 2: KubeVirt-based Hub Architecture Greenfield (Special Case)

When you have a **single large bare metal machine**, you can consolidate the Global Hub
and the ACM spoke hubs (Hub 1..N) onto one physical host using OpenShift Virtualization (KubeVirt).

### What changes

The physical cluster count drops to one. ACM Hub 1..N run as OCP clusters
**inside KubeVirt VMs** on the same physical machine as the Global Hub.
From AutoShift's perspective the logical hub-of-hubs topology is identical —
it only sees clusters, not what runs underneath them.

### Deployment sequence

```
Step 1: Install OCP on the large bare metal machine → Cluster 0 (ACM Global Hub)

  ┌──────────────────────────────────────────────────────────────────┐
  │  Large Bare Metal Machine                                        │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  Cluster 0 — Host OCP Cluster                              │  │
  │  │  - ACM Global Hub                                          │  │
  │  │  - OpenShift GitOps + AutoShift v2                         │  │
  │  │  - OpenShift Virtualization (KubeVirt)                     │  │
  │  └────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘


Step 2: Use OpenShift Virtualization to create 6 VMs (3 per hub as an example)

  ┌──────────────────────────────────────────────────────────────────┐
  │  Cluster 0 (host)                                                │
  │                                                                  │
  │   VM1 ┐                      VM4 ┐                              │
  │   VM2 ├─ will become Hub 1   VM5 ├─ will become Hub 2           │
  │   VM3 ┘                      VM6 ┘                              │
  └──────────────────────────────────────────────────────────────────┘


Step 3: Install OCP on each group of 3 VMs

  Options (see FakeFish section below):
  - Agent-based installer ISO  ← manual, no FakeFish needed
  - HyperShift + KubeVirt provider  ← automated, no FakeFish needed
  - ACM Assisted Installer + Metal3  ← automated, FakeFish REQUIRED


Step 4: Import Hub 1 and Hub 2 into ACM Global Hub as ManagedClusters

  ┌──────────────────────────────────────────────────────────────────┐
  │  Cluster 0 (host) — ACM Global Hub                               │
  │                                                                  │
  │  ┌──────────────────────┐   ┌──────────────────────┐            │
  │  │  ACM Hub 1           │   │  ACM Hub 2           │            │
  │  │  (3 KubeVirt VMs)    │   │  (3 KubeVirt VMs)    │            │
  │  └──────────────────────┘   └──────────────────────┘            │
  └──────────────────────────────────────────────────────────────────┘


Step 5: AutoShift v2 manages Hub 1 and Hub 2 from the Global Hub (same as Pattern 1)
```

### Full topology (physical view)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Large Bare Metal Machine (L0)                                              │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  Cluster 0 — Host OCP + ACM Global Hub (L1)                          │  │
│  │                                                                       │  │
│  │   ┌─────────────────────────┐   ┌─────────────────────────┐          │  │
│  │   │  ACM Hub 1              │   │  ACM Hub 2              │          │  │
│  │   │  (3 KubeVirt VMs)       │   │  (3 KubeVirt VMs)       │          │  │
│  │   │                         │   │                         │          │  │
│  │   │  Metal3 + AI Operator   │   │  Metal3 + AI Operator   │          │  │
│  │   └────────────┬────────────┘   └──────────────┬──────────┘          │  │
│  └────────────────┼──────────────────────────────┬─┼────────────────────┘  │
└───────────────────┼──────────────────────────────┼─┼───────────────────────┘
                    │                              │ │
                    │   OOB / BMC Network          │ │
                    │   (Redfish / IPMI)           │ │
                    ▼                              ▼ ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Physical Bare Metal Target Servers                                         │
│  (iDRAC / iLO / Redfish BMCs)                                              │
│                                                                             │
│   [Spoke A]        [Spoke B]        [Spoke C]        [Spoke D]             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Metal3 works on a KubeVirt-based hub cluster

Even though the ACM Hub cluster runs inside VMs, the Bare Metal Operator (Metal3) inside
ACM does not care whether the hub is physical or virtual. It only needs L2/L3 network
connectivity from the hub cluster to the out-of-band management interfaces of the physical
target servers.

#### Critical network and infrastructure requirements

**BMC network accessibility**

The KubeVirt VMs hosting the ACM Hub must have direct L3 network access to the
Redfish/iDRAC/iLO management IPs of the physical target servers. Metal3 communicates
over HTTP/HTTPS to authenticate with the BMCs and trigger virtual media boots.

```
KubeVirt VM (Metal3 pod)
        │
        │  HTTPS (port 443) — Redfish API
        ▼
iDRAC / iLO / Redfish BMC  ──►  Physical target server
```

**Multus secondary network (L2 provisioning network)**

If you are using the Bare Metal Operator with Ironic inspection (rather than the
agent-based Assisted Installer), the VMs require a secondary network interface via
Multus attached directly to the physical VLAN where target nodes perform PXE/DHCP boots.

```
KubeVirt VM
├── Primary NIC  ─── cluster SDN (normal pod traffic)
└── Secondary NIC (Multus NetworkAttachmentDefinition)
        │
        └── Physical provisioning VLAN (PXE / DHCP / TFTP)
                │
                ▼
        Target bare metal nodes (boot NIC)
```

**Assisted Installer service endpoints**

When using the Assisted Installer (the standard ACM workflow), target physical nodes
boot a discovery ISO, fetch instructions, and download RHCOS images. The target nodes
must be able to reach the ACM Hub's `assisted-service` route over port 443 or 80.

```
Physical target node (booted from discovery ISO)
        │
        │  HTTPS port 443
        ▼
assisted-service route on ACM Hub  ──►  registers node, drives install
```

---

## Provisioning methods compared

### Agent-based ISO vs ACM Assisted Installer + Metal3

These are two distinct approaches to provisioning a cluster. The key difference is **who
handles the ISO attachment and host boot** — you (or your automation) or Metal3.

"Manual" does not mean physical access to machines. Every step in the agent-based ISO
flow can be done programmatically via APIs, with no one touching the hardware.

```
Agent-based ISO ("manual"):
  Your script / automation
        ├── generates ISO (openshift-install CLI or Assisted Installer REST API)
        ├── attaches ISO  (KubeVirt API patch, or Redfish VirtualMedia call directly to iDRAC/iLO)
        └── boots host    (KubeVirt virtctl / Redfish ComputerSystem.Reset)
  ACM's Assisted Installer (running on the hub) guides the install from here.
  In disconnected environments: nodes pull RHCOS and release images from
  internal mirror registry, never touching the internet.

ACM Assisted Installer + Metal3 (automated):
  Metal3
        ├── reads BareMetalHost resource (BMC address, credentials)
        ├── calls Redfish VirtualMedia to mount the discovery ISO
        └── calls Redfish to power-cycle and boot the host
  ACM's Assisted Installer (running on the hub) guides the install from here.
  In disconnected environments: nodes pull RHCOS and release images from
  internal mirror registry, never touching the internet.
```

| | Agent-based ISO | ACM AI + Metal3 |
|---|---|---|
| Who mounts the ISO? | Your code (KubeVirt API or Redfish API directly) | Metal3 via Redfish |
| Who power-cycles the host? | Your code | Metal3 via Redfish |
| BareMetalHost resource needed? | No | Yes |
| Works on KubeVirt VMs without FakeFish? | Yes | No — VMs have no real BMC |
| Works on real bare metal? | Yes | Yes |
| Fully remote / no physical access needed? | Yes | Yes |
| Works in disconnected / air-gapped? | Yes | Yes |

### Disconnected / on-premises considerations

In a disconnected environment, the Assisted Installer that ships inside ACM
(`assisted-service` pod on the hub, enabled via `MultiClusterHub`) is the correct
component to use. No external or standalone Assisted Installer is needed.

The flow is the same as connected, but all external endpoints are replaced with
internal infrastructure:

```
Connected:
  Target node boots ISO
        ├── pulls RHCOS from mirror.openshift.com
        └── pulls release images from quay.io / registry.redhat.io

Disconnected:
  Target node boots ISO
        ├── pulls RHCOS from internal HTTP server (no internet)
        └── pulls release images from internal mirror registry (no internet)
```

To configure ACM's Assisted Installer for disconnected:

| What | Where it is configured |
|---|---|
| RHCOS ISO location | `AgentServiceConfig.spec.osImages` — URL of internally hosted ISO |
| Mirror registry CA | Injected into `AgentServiceConfig`, baked into discovery ISO via `InfraEnv` |
| Image redirects | `ImageDigestMirrorSet` (IDMS) or `ImageContentSourcePolicy` (ICSP) on target nodes |
| Pull secret | Must include credentials for the internal mirror registry |

AutoShift handles all of the above through the `disconnected:` block in cluster values
files — it wires up `InfraEnv`, `AgentServiceConfig`, IDMS/ICSP, and mirror catalog
sources automatically. See the
[cluster provisioning docs](https://auto-shift.github.io/autoshiftv2/cluster-install/)
for the full `disconnected:` config reference.

**For KubeVirt VMs specifically**, the agent-based ISO approach is straightforward
because you can attach ISOs and control boot directly through the KubeVirt API:

```
Generate ISO  ─►  oc apply DataVolume (ISO)
                        │
                        ▼
              Patch VirtualMachine spec (cdrom: ISO DataVolume)
                        │
                        ▼
              virtctl start <vm>  (or patch spec.running: true)
                        │
                        ▼
              Watch AgentClusterInstall status
```

---

## Redfish emulation for VMs — FakeFish and sushy-emulator

### Two different tools, same job

When Metal3 needs to provision a KubeVirt VM, it expects a Redfish BMC endpoint to
call — but VMs have no physical BMC. Two separate tools solve this by emulating the
Redfish API:

| Tool | Project | Backend | Origin |
|---|---|---|---|
| **sushy-emulator** | OpenStack / sushy-tools | libvirt | OpenStack Ironic ecosystem |
| **FakeFish** | Standalone | KubeVirt API | Metal3 / KubeVirt community |

They are **not the same project**. sushy-emulator is part of the OpenStack sushy-tools
package and targets libvirt. FakeFish is a separate tool built specifically for
KubeVirt, making it the more natural fit for OpenShift Virtualization environments.
Both translate Metal3's Redfish calls into VM control operations.

```
Without emulation (real bare metal):
  Metal3  ──Redfish──►  iDRAC / iLO / BMC  ──►  Physical Server

With sushy-emulator (libvirt VMs):
  Metal3  ──Redfish──►  sushy-emulator  ──libvirt API──►  VM

With FakeFish (KubeVirt VMs):
  Metal3  ──Redfish──►  FakeFish  ──KubeVirt API──►  KubeVirt VM
```

### Production readiness

**Neither FakeFish nor sushy-emulator is production-grade.** Both are lab and
development tools. Red Hat does not support either for production deployments.

For production KubeVirt-based hub clusters, the supported path is
**HyperShift + KubeVirt provider**, which has first-class ACM and Red Hat support
and requires no BMC emulation:

| Approach | Use case | Production? |
|---|---|---|
| FakeFish | Lab / dev / CI — ACM AI + Metal3 driving KubeVirt VMs | No |
| sushy-emulator | Lab / dev / CI — ACM AI + Metal3 driving libvirt VMs | No |
| HyperShift + KubeVirt provider | Production hosted clusters on KubeVirt | Yes |
| Agent-based ISO (scripted) | Any environment, including production | Yes |

### When is a Redfish emulator needed?

| What you are provisioning | Method | Emulator needed? |
|---|---|---|
| Hub 1..N on KubeVirt VMs — agent-based ISO | Scripted, no Metal3 | No |
| Hub 1..N on KubeVirt VMs — HyperShift + KubeVirt provider | HyperShift | No |
| Hub 1..N on KubeVirt VMs — ACM Assisted Installer + Metal3 | Automated | **Yes** (FakeFish or sushy-emulator) |
| Spoke clusters on real bare metal (from Hub 1..N) | ACM AI + Metal3 | No |
| Spoke clusters on AWS / vSphere (from Hub 1..N) | Hive | No |
| Spoke clusters as KubeVirt VMs (from Hub 1..N) | ACM AI + Metal3 | **Yes** (FakeFish or sushy-emulator) |

### Network requirements when using a Redfish emulator

The emulator must be reachable from Metal3 pods, and must itself be able to reach
the KubeVirt API. Each VM gets a virtual BMC address that the corresponding
`BareMetalHost` resource points to.

```
┌────────────────────────────────────────────────────────────────┐
│  Cluster 0 (host)                                              │
│                                                                │
│  ┌────────────────────────┐                                    │
│  │  Metal3 pod            │──Redfish calls──►  FakeFish pod    │
│  └────────────────────────┘                        │           │
│                                                    │           │
│                                         KubeVirt API calls     │
│                                                    │           │
│                                                    ▼           │
│                                           KubeVirt VM          │
│                                    (power on/off, ISO mount)   │
└────────────────────────────────────────────────────────────────┘
```

The emulator must be:
- Reachable from Metal3 pods on the BMC network (or a dedicated service network)
- Able to reach the KubeVirt API
- Configured with a virtual BMC address per VM for `BareMetalHost` resources to reference

---

## Spoke Cluster Provisioning from Hub 1..N

Once Hub 1 and Hub 2 are running ACM and registered with the Global Hub,
they provision their own spoke clusters. AutoShift v2 applies here unchanged.

```
Hub 1 (KubeVirt-based, but ACM sees it as a normal cluster)
│
├── Spoke: real bare metal  ──  Metal3 + Assisted Installer → real Redfish BMC  (no FakeFish)
├── Spoke: AWS              ──  Hive ClusterDeployment                           (no FakeFish)
├── Spoke: vSphere          ──  Hive ClusterDeployment                           (no FakeFish)
└── Spoke: KubeVirt VM      ──  Metal3 + Assisted Installer → FakeFish           (FakeFish needed)
```

---

## Summary

| Scenario | Separate physical clusters? | FakeFish needed? | AutoShift v2 supports? |
|---|---|---|---|
| Pattern 1: standard greenfield HoH | Yes | No | Yes, fully |
| Pattern 2: all hubs on one large BM machine via KubeVirt | No — VMs on one host | Only if using ACM AI + Metal3 to provision the VMs | Yes, once hubs are imported |
| Hub 1..N provisioning real BM spokes | n/a | No | Yes |
| Hub 1..N provisioning AWS/vSphere spokes | n/a | No | Yes |
| Hub 1..N provisioning KubeVirt VM spokes | n/a | Yes + network config | Yes (after FakeFish wired up) |

The logical hub-of-hubs model is identical in both patterns.
The physical consolidation in Pattern 2 is transparent to AutoShift and ACM policy machinery.
