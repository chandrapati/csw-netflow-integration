# Cisco Secure Workload — NetFlow Integration Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Use Cases](#3-use-cases)
4. [Prerequisites](#4-prerequisites)
5. [Step A — Configure NetFlow on Cisco Switches/Routers](#5-step-a--configure-netflow-on-cisco-switchesrouters)
6. [Step B — Configure the NetFlow Connector on CSW](#6-step-b--configure-the-netflow-connector-on-csw)
7. [Supported IPFIX/NetFlow Elements](#7-supported-ipfixnetflow-elements)
8. [Rate Limiting](#8-rate-limiting)
9. [Verification](#9-verification)
10. [Limits](#10-limits)
11. [Troubleshooting](#11-troubleshooting)
12. [Related Resources](#12-related-resources)

---

## 1. Overview

The **NetFlow connector** allows Cisco Secure Workload to ingest flow observations from **Cisco routers and switches** without deploying CSW software agents on the monitored hosts. Switches aggregate traffic into flows and export them via **NetFlow v9 or IPFIX** to the NetFlow connector on a **CSW Ingest appliance**.

This is the primary agentless flow ingestion path for physical and virtual workloads behind Cisco network devices — particularly valuable for:
- **Legacy servers** that cannot run a CSW agent
- **Mainframe / bare-metal** environments
- **Network devices** themselves (visibility into routing/switching traffic)
- **IoT / OT devices** without agent support

> **Uses Ingest appliance** (not Edge appliance — unlike ISE, AnyConnect, and ServiceNow connectors).

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Data Center / Campus Network                       │
│                                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────┐ │
│  │  Cisco IOS/     │  │  Cisco NX-OS    │  │  Cisco IOS-XE/XR     │ │
│  │  IOS-XE Router  │  │  Nexus Switch   │  │  Campus Switch       │ │
│  │                 │  │                 │  │                       │ │
│  │  NetFlow v9 /   │  │  NetFlow v9 /   │  │  NetFlow v9 / IPFIX  │ │
│  │  IPFIX enabled  │  │  IPFIX enabled  │  │  enabled              │ │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬────────────┘ │
│           │                   │                       │              │
│           └───────────────────┴───────────────────────┘              │
│                               │ NetFlow v9/IPFIX (UDP)               │
│                               ▼                                      │
│             ┌─────────────────────────────────────┐                  │
│             │    CSW Ingest Appliance              │                  │
│             │    (NetFlow Connector)               │                  │
│             │                                     │                  │
│             │  Listening: UDP port 2055 (default) │                  │
│             │  Max rate: 15,000 flows/second       │                  │
│             └───────────────┬─────────────────────┘                  │
│                             │ Forwarded to CSW collectors            │
│             ┌───────────────▼─────────────────────┐                  │
│             │  Cisco Secure Workload Cluster       │                  │
│             │  Flow analysis, ADM, policy          │                  │
│             └─────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Use Cases

### Use Case 1 — Agentless Server Visibility
Legacy servers (Windows 2003, old RHEL, mainframes) can't run a CSW agent. NetFlow from the top-of-rack switch provides full east-west flow visibility for those workloads.

### Use Case 2 — Data Center Fabric Visibility
Enable NetFlow on Nexus 9000 switches in the data center fabric. CSW builds an application dependency map from actual switch-observed flows — no agent deployment required on any server.

### Use Case 3 — Perimeter and WAN Visibility
Configure NetFlow on edge routers to capture ingress/egress flows for north-south traffic analysis. Useful for identifying external connections to internal workloads.

### Use Case 4 — Hybrid Agent + NetFlow
Deploy CSW agents where possible (modern Linux/Windows servers) and use NetFlow for the remaining agentless workloads. CSW correlates both sources for complete coverage.

---

## 4. Prerequisites

### Network device requirements
- [ ] Cisco IOS, IOS-XE, IOS-XR, NX-OS, or any device supporting **NetFlow v9 or IPFIX**
- [ ] NetFlow/IPFIX feature licensed and enabled on the device
- [ ] Network reachability: switch → CSW Ingest appliance on **UDP port 2055** (or configured port)

### CSW / infrastructure requirements
- [ ] **CSW Ingest appliance** deployed and registered (not Edge — NetFlow uses Ingest)
- [ ] Ingest appliance has sufficient capacity for the expected flow rate (max 15,000 flows/second per connector)
- [ ] VRF configuration: Ingest appliance VRF assignment covers the subnet of the monitored workloads

---

## 5. Step A — Configure NetFlow on Cisco Switches/Routers

### IOS / IOS-XE (recommended: Flexible NetFlow)

```ios
! Step 1: Create a flow record defining what to track
flow record CSW-FLOW-RECORD
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 match ip protocol
 match ip tos
 collect transport tcp flags
 collect interface input
 collect interface output
 collect counter bytes
 collect counter packets
 collect timestamp sys-uptime first
 collect timestamp sys-uptime last

! Step 2: Create a flow exporter pointing to CSW Ingest appliance
flow exporter CSW-EXPORTER
 destination 10.x.x.x          ! IP of CSW Ingest appliance
 source GigabitEthernet0/0     ! Source interface
 transport udp 2055             ! Default NetFlow port
 export-protocol netflow-v9
 template data timeout 60

! Step 3: Create a flow monitor
flow monitor CSW-MONITOR
 record CSW-FLOW-RECORD
 exporter CSW-EXPORTER
 cache timeout active 60
 cache timeout inactive 15

! Step 4: Apply to interfaces (both directions)
interface GigabitEthernet0/1
 ip flow monitor CSW-MONITOR input
 ip flow monitor CSW-MONITOR output
```

### NX-OS (Cisco Nexus)

```nxos
! Enable NetFlow feature
feature netflow

! Create exporter
flow exporter CSW-EXPORT
  destination 10.x.x.x use-vrf management
  source mgmt0
  transport udp 2055
  version 9
    template data timeout 60

! Create flow record
flow record CSW-RECORD
  match ipv4 source address
  match ipv4 destination address
  match transport source-port
  match transport destination-port
  match ip protocol
  collect transport tcp flags
  collect counter bytes
  collect counter packets
  collect timestamp sys-uptime first
  collect timestamp sys-uptime last
  collect interface input
  collect interface output

! Create flow monitor
flow monitor CSW-MONITOR
  record CSW-RECORD
  exporter CSW-EXPORT
  cache timeout active 60
  cache timeout inactive 15

! Apply to VLAN interface
interface Vlan100
  ip flow monitor CSW-MONITOR input
  ip flow monitor CSW-MONITOR output
```

### Verify NetFlow exports are being sent
```
! IOS/IOS-XE:
show flow exporter CSW-EXPORTER statistics
show flow monitor CSW-MONITOR cache

! NX-OS:
show flow exporter CSW-EXPORT
show flow monitor CSW-MONITOR cache
```

---

## 6. Step B — Configure the NetFlow Connector on CSW

### B1 — Navigate to connector configuration

1. CSW UI: **Manage > Virtual Appliances**
2. Select the **Ingest appliance**
3. **Connectors** tab → **+ Add Connector** → **NetFlow**

### B2 — Connector settings

| Field | Value |
|-------|-------|
| **Connector Name** | e.g., `netflow-dc-nexus` |
| **Connector Type** | NetFlow |
| **Listening Port** | UDP 2055 (default; must match switch exporter config) |

### B3 — VRF assignment

Configure the VRF for this connector:
1. **Manage > Workloads > Agents > Configuration tab**
2. **Agent Remote VRF Configurations > Create Config**
3. Provide:
   - VRF Name (the VRF the monitored workloads belong to)
   - IP subnet CIDR of the monitored workloads
   - Port range: `2055-2055`

> One NetFlow connector per VRF. If you have multiple VRFs, create separate connectors.

### B4 — Apply configuration
Click **Test and Apply**.

---

## 7. Supported IPFIX/NetFlow Elements

The NetFlow connector supports a specific set of IPFIX information elements. Key mandatory fields:

| Element | Description | Required? |
|---------|-------------|---------|
| `octetDeltaCount` (ID 1) | Bytes in flow | Yes |
| `packetDeltaCount` (ID 2) | Packets in flow | Yes |
| `sourceIPv4Address` (ID 8) | Source IP | Yes |
| `destinationIPv4Address` (ID 12) | Destination IP | Yes |
| `sourceTransportPort` (ID 7) | Source port | Yes |
| `destinationTransportPort` (ID 11) | Destination port | Yes |
| `protocolIdentifier` (ID 4) | IP protocol | Yes |
| `flowStartSysUpTime` (ID 22) | Flow start time | Yes |
| `flowEndSysUpTime` (ID 21) | Flow end time | Yes |
| `tcpControlBits` (ID 6) | TCP flags | Recommended |
| `ingressInterface` (ID 10) | Ingress interface | Recommended |
| `egressInterface` (ID 14) | Egress interface | Recommended |

---

## 8. Rate Limiting

| Metric | Value |
|--------|-------|
| Maximum flows per second | **15,000** |
| Behavior when exceeded | Additional flows dropped |
| Recommended action if exceeded | Reduce source interfaces; increase sampling rate; add second Ingest appliance |

> Monitor flow rate: **CSW UI > Agents > [NetFlow Connector] > Deep Visibility Agent > Packet Rate graph**

---

## 9. Verification

### Check connector status
**Manage > Virtual Appliances > [Ingest] > Connectors**
NetFlow connector should show **Status: Active**

### Check flow data in CSW
1. **Observe > Traffic** → filter by the subnet behind the switch
2. Confirm flows are appearing with src/dst IPs matching expected workloads

### Check on the switch
```ios
show flow exporter CSW-EXPORTER statistics
! Look for: "Packets sent to destination" incrementing
```

### Check on Ingest appliance
```bash
tcpdump -i any -n port 2055
# Should show UDP packets from switch management IPs
```

---

## 10. Limits

| Metric | Limit |
|--------|-------|
| Flow rate per connector | 15,000 flows/second |
| NetFlow versions supported | v9 and IPFIX only (not v5) |
| VRF per connector | 1 |
| Connector location | Ingest appliance (not Edge) |

---

## 11. Troubleshooting

| Symptom | Check |
|---------|-------|
| No flows in CSW | Verify switch exporter destination IP/port matches Ingest appliance; check UDP 2055 is open in any firewall between switch and Ingest; confirm VRF assignment covers the subnet |
| Flows appear but workloads not labeled | Add CSW agents or ServiceNow labels for workload context — NetFlow provides flow data only, not host metadata |
| High drop rate on connector | Flow rate exceeding 15,000/second; reduce NetFlow source interfaces or increase sampling interval on switch |
| Template not received | Ensure switch exports templates regularly (`template data timeout 60`); verify IPFIX/v9 (not v5) is configured |

---

## 12. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [csw-erspan-integration](https://github.com/chandrapati/csw-erspan-integration) | ERSPAN agentless packet capture for deeper visibility | Full packet inspection |
| [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL flow ingestion from Cisco Secure Firewall | Firewall-specific flows |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents for deep process-level visibility | Agent-based workloads |
| [csw-servicenow-integration](https://github.com/chandrapati/csw-servicenow-integration) | ServiceNow CMDB labels for workload context | Enriching agentless workloads |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | ADM → policy workflow | Building policy from flows |

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
