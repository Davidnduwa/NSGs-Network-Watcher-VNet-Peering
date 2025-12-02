NSGs, Network Watcher & VNet Peering

This lab provides essential hands-on experience with Network Security Groups (**NSGs**) logic, **ICMP** behavior, the **Network Watcher** troubleshooting tool, and the fundamental constraints of **VNet Peering**.

## 🎯 Lab Goals

  * Build and test **NSG evaluation logic** (priority and specificity).
  * Understand the difference between **TCP/UDP** filtering and **ICMP** behavior.
  * Use **Network Watcher's "Connection Troubleshoot"** tool for diagnostics.
  * Validate the requirement for **non-overlapping address spaces** in VNet peering.
  * Understand basic subnet-to-subnet traffic flows within a VNet.

-----

## 🛠️ Environment Setup

We will start with a single Virtual Network segmented into two subnets to test NSG rules and intra-VNet communication.

### Virtual Network

| Resource | Address Space |
| :--- | :--- |
| **VNet** | `vnet-nsg-lab` (`10.0.0.0/16`) |
| **Subnet A** | `subnet-a` (`10.0.1.0/24`) |
| **Subnet B** | `subnet-b` (`10.0.2.0/24`) |

### Virtual Machines

| VM Name | Subnet | Notes |
| :--- | :--- | :--- |
| **vm-a** | `subnet-a` | Source for connectivity tests (Default OS behavior) |
| **vm-b** | `subnet-b` | Destination for connectivity tests **(Must enable ICMP Echo in Windows Firewall)** |

> **Critical Note:** Windows Server OS disables **ICMP Echo** (Ping) by default in the host firewall. Ping will fail until you specifically enable the **"File and Printer Sharing (Echo Request - ICMPv4-In)"** rule on `vm-b`.

-----

## 🔐 NSG Configuration and Evaluation

We will create an NSG (`nsg-b`) and associate it with **subnet-b** to control inbound traffic to `vm-b`.

### NSG Rules: `nsg-b` (Inbound)

| Priority | Source | Destination | Protocol | Port | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **300** | `10.0.1.0/24` (Subnet A) | `10.0.2.0/24` (Subnet B) | **TCP** | `*` | **Allow** | Allow specific subnet-to-subnet TCP traffic. |
| **310** | `Any` | `10.0.2.0/24` (Subnet B) | **TCP** | `*` | **Deny** | Deny all other TCP traffic to Subnet B. |

### 🔎 Interpretation

  * Traffic is evaluated by **Priority** (lowest number is highest priority).
  * Traffic from **Subnet A** (`10.0.1.0/24`) to **Subnet B** (`10.0.2.0/24`) is **Allowed** by Rule 300.
  * All other **TCP** traffic is caught and **Denied** by the broader Rule 310.
  * **Crucially:** These rules only specify **TCP**. **ICMP** (Ping) traffic is **NOT** affected by either rule, and will bypass them to be handled by the default rules (which generally allow intra-VNet traffic).

-----

## 🔍 Connectivity Tests

Perform the following tests from `vm-a` (in `subnet-a`) to `vm-b` (in `subnet-b`).

### Test 1: ICMP (Ping)

```powershell
ping <vm-b-private-ip>
```

| Expected Result | Reason |
| :--- | :--- |
| **✔️ Works** | ICMP is not filtered by the TCP rules (300, 310). Since `vm-b`'s Windows Firewall was configured to allow Echo, the traffic flows based on the **Default NSG rule** allowing VNet-to-VNet communication. |

### Test 2: TCP (e.g., Port 443)

This simulates an attempt to connect to a service on a TCP port.

```powershell
Test-NetConnection <vm-b-private-ip> -Port 443
```

| Expected Result | Reason |
| :--- | :--- |
| **Likely Fails/Warns** | **Rule 300** allows the TCP connection at the NSG level. However, if no application is actively listening on **port 443** on `vm-b`, the connection will still fail. |

> **Demonstration:** This shows the difference between **Network Reachability** (NSG allows it) and **Application Availability** (Is a service running to answer the request?).

-----

## 🛰️ Network Watcher Diagnostics

Use the **Connection Troubleshoot** tool in Azure Network Watcher to validate the NSG and routing configuration without relying on an application listener.

1.  Enable **Network Watcher** for the region containing your VNet.
2.  Go to **Network Watcher → Connection Troubleshoot**.

### Test 1: TCP (Port 443)

| Setting | Value |
| :--- | :--- |
| **Source** | `vm-a` |
| **Destination** | `vm-b`: **Port 443** |
| **Expected Result** | **Succeeds (Green)** at the **NSG** level due to Rule 300. |

### Test 2: ICMP

| Setting | Value |
| :--- | :--- |
| **Source** | `vm-a` |
| **Destination** | `vm-b`: **ICMP** |
| **Expected Result** | **Succeeds (Green)** at the **NSG** level because the traffic bypasses the TCP rules. |

> **Demonstrates:** Network Watcher confirms the NSG rules are correct. If the actual `ping` test fails, you know the issue lies with the **Windows Firewall** on the VM, not the Azure NSG.

-----

## 🌐 VNet Peering Scenarios

VNet Peering is the mechanism used to extend your private Azure network across VNets. It has one strict requirement.

### ❌ Scenario A: Overlapping Address Spaces (Peering Fails)

| VNet | Address Space |
| :--- | :--- |
| `vnet-overlap1` | `10.1.0.0/16` |
| `vnet-overlap2` | `10.1.0.0/17` |

  * **Action:** Attempt to create peering.
  * **Result:** Azure will **refuse to peer** these networks.
  * **Reason:** Overlapping ranges lead to **routing ambiguity**; the system wouldn't know which VNet a packet destined for `10.1.1.0` should go to.

### ✔️ Scenario B: Non-Overlapping VNets (Peering Succeeds)

| VNet | Address Space |
| :--- | :--- |
| `vnet-ok1` | `10.12.0.0/16` |
| `vnet-ok2` | `172.16.0.0/17` |

  * **Action:** Create peering from `ok1` → `ok2` AND `ok2` → `ok1`.
  * **Result:** The peering status becomes **Connected**.
  * **Outcome:** Traffic flows seamlessly between the VNets using **private IP addresses**, extending the private network boundary.

-----

## 🧠 Key Takeaways

| Topic | Key Principles |
| :--- | :--- |
| **NSG Rules** | **Priority** (Lower number = higher priority) is evaluated first. **Evaluation stops at the first match** (either Allow or Deny). |
| **Protocol Filtering** | A rule set to **TCP** will **not** filter **ICMP**. To block ICMP, the protocol must be set to **ICMP** or **Any**. |
| **Windows Firewall** | This is the final layer of defense **inside the VM**. It must be configured to allow protocols like **ICMP Echo** for internal ping tests to succeed. |
| **Network Watcher** | The **"Connection Troubleshoot"** tool provides a clear, objective analysis of whether **NSG rules** or **routing** are the cause of a failure. |
| **VNet Peering** | **Requires non-overlapping address ranges** to prevent routing conflicts. Peering successfully extends the private network. |

-----
