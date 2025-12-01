NSGs, Network Watcher & VNet Peering

This lab provides hands-on experience with Network Security Groups (NSGs), ICMP behavior, Network Watcher troubleshooting, and VNet peering constraints.

🎯 Goal of This Lab

✔ Build and test NSG evaluation logic
✔ Understand TCP vs ICMP behavior
✔ Use Network Watcher's “Connection troubleshoot” tool
✔ Validate overlapping vs non-overlapping VNet peering
✔ Understand subnet-to-subnet traffic flows inside a VNet

🛠️ Lab Environment
Virtual Network
Resource	Address Space
vnet-nsg-lab	10.0.0.0/16
subnet-a	10.0.1.0/24
subnet-b	10.0.2.0/24
Virtual Machines
VM	Subnet	Notes
vm-a	subnet-a	Source for tests
vm-b	subnet-b	Enable ICMP Echo in Windows Firewall

ICMP is disabled by default on Windows - this is WHY ping doesn’t work until you turn it on.


🔐 NSG Configuration: nsg-b

Associate nsg-b to subnet-b.

Inbound Rules
Priority	Source	Destination	Protocol	Action
300	10.0.1.0/24	10.0.2.0/24	TCP	Allow
310	Any	10.0.2.0/24	TCP	Deny
🔎 Interpretation

Traffic from subnet-a → subnet-b (TCP) is allowed because rule 300 is a more specific Allow.

All other TCP traffic → subnet-b is denied by rule 310.

ICMP is NOT affected by these rules because NSGs only inspect TCP/UDP unless protocol is set to “Any”.

🔍 Connectivity Tests
From vm-a:
✔ Ping vm-b
ping <vm-b-private-ip>


Result: Works, because ICMP is not filtered by TCP rules.

✔ Test TCP port
Test-NetConnection <vm-b-private-ip> -Port 443


Result: Likely fails, because:

No listener → no response

Or NSG denies it unless an app is listening on 443

This demonstrates the difference between network reachability and application availability.

🛰️ Network Watcher - Connection Troubleshoot

Enable Network Watcher if required:

Azure Portal → Network Watcher → “Enable”

Test 1: TCP (443)

Source: vm-a
Destination: vm-b:443
Expected: Fail or Warn depending on listener + NSG

Test 2: ICMP

Source: vm-a
Destination: vm-b
Expected: Success

Demonstrates:

TCP is evaluated by NSG rules

ICMP requires Windows Firewall to allow it

NSGs do not filter ICMP unless explicitly set to “Any”

🌐 VNet Peering Scenarios
❌ Overlapping Address Spaces - Peering Fails
VNet	Address Space
vnet-overlap1	10.1.0.0/16
vnet-overlap2	10.1.0.0/17

Azure will refuse to peer these networks.

Reason: Overlaps → routing ambiguity.

✔ Non-Overlapping VNets - Peering Succeeds
VNet	Address Space
vnet-ok1	10.12.0.0/16
vnet-ok2	172.16.0.0/17

Create peering both ways:

ok1 → ok2

ok2 → ok1

Traffic flows seamlessly via private IPs.

🧠 Key Takeaways
🔹 NSG Rules

Evaluated by priority (lower number = higher priority)

Evaluation stops at first match

TCP rules do not affect ICMP

🔹 Windows Firewall

Must enable ICMP Echo for ping to work

🔹 Network Watcher

“Connection troubleshoot” simulates actual packet flows

Helps confirm:

NSG behavior

Routing

Firewall issues

🔹 VNet Peering

Requires non-overlapping ranges

Overlaps → automatic failure

Peering = private network extension
