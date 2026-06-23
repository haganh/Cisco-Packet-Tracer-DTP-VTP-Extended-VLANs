# Cisco Packet Tracer: DTP, VTP, and Extended VLANs

This repository documents the implementation of dynamic trunking negotiation (DTP), centralized database synchronization via VLAN Trunking Protocol (VTP), and scaling VLAN database boundaries with Extended VLANs in transparent mode across a multi-switch loop topology.

## Summary

This lab builds on core VLAN segmentation by automating VLAN management across a 3-switch topology. A VTP Server/Client architecture is used to centrally propagate VLAN changes, DTP is used to dynamically negotiate a trunk link, and VTP Transparent mode is configured to support Extended-range VLANs beyond the standard database limit. Connectivity is validated end-to-end with targeted ping tests confirming both expected VLAN isolation and expected reachability, demonstrating an understanding of Layer 2/Layer 3 boundaries and the ability to troubleshoot and verify a multi-switch configuration.

## Network Topology

<img width="400" height="286" alt="Screenshot 2026-06-22 230008" src="https://github.com/user-attachments/assets/ea8165c5-8202-4d41-912b-834bc6399319" />


---

## Addressing Table

| Device | Interface | IP Address | Subnet Mask | Default Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **S1** | VLAN 99 | `192.168.99.1` | `255.255.255.0` | N/A |
| **S2** | VLAN 99 | `192.168.99.2` | `255.255.255.0` | N/A |
| **S3** | VLAN 99 | `192.168.99.3` | `255.255.255.0` | N/A |
| **PC-A** | NIC | `192.168.10.1` | `255.255.255.0` | N/A |
| **PC-B** | NIC | `192.168.20.1` | `255.255.255.0` | N/A |
| **PC-C** | NIC | `192.168.10.2` | `255.255.255.0` | N/A |

---

## Lab Objectives & Key Technologies

* **VTP Domain Synchronization:** Establish a centralized database architecture using S2 as a VTP Server and S1/S3 as VTP Clients to automate VLAN propagation.
* **Dynamic Trunking Protocol (DTP):** Implement Dynamic Desirable negotiation on S1 to dynamically establish trunking links with neighboring switches.
* **VLAN Segmentation:** Isolate traffic by configuring standard VLAN scopes (VLAN 10, 20, 30, and 99) and assigning access ports.
* **Extended VLAN Capabilities:** Transition S1 into VTP Transparent mode to bypass VTP database limits and configure an extended-range VLAN (VLAN 2000).

---

## Step-by-Step Implementation Guide & Command Log

### Step 1: VTP Domain & Server/Client Setup

We configure **S2** as the master database controller (VTP Server), and configure **S1** and **S3** to receive and execute VLAN instructions dynamically (VTP Clients).

####  Switch S2 (Server) Configuration:

```text
S2# configure terminal
S2(config)# vtp domain Nerdify
S2(config)# vtp mode server
S2(config)# vtp password cisco
```

**What is happening:** S2 is now authorized to create, modify, and delete VLANs. Setting the domain to Nerdify and password to cisco creates a secure replication group.

####  Switch S1 (Client) Configuration:

```text
S1# configure terminal
S1(config)# vtp domain Nerdify
S1(config)# vtp mode client
S1(config)# vtp password cisco
```

####  Switch S3 (Client) Configuration:

```text
S3# configure terminal
S3(config)# vtp domain Nerdify
S3(config)# vtp mode client
S3(config)# vtp password cisco
```

**What is happening:** S1 and S3 lock themselves down to "Read-Only" mode for VLAN databases. They will monitor incoming trunk ports for VTP frame announcements belonging to the Nerdify domain.

---

### Step 2: VTP Verification

 **Verification Command (All Switches):**

```text
S2# show vtp status
```

**What is happening:** This output displays important VTP parameters. Pay attention to the Configuration Revision Number (currently 0). This counter increments every time a VLAN is added or modified. The client with a lower number will instantly clone the database of any server with a higher number!

---

### Step 3: Negotiating Trunks via DTP (Dynamic Trunking Protocol)

Instead of statically configuring trunk modes on both sides of S1 and S2, we leverage Cisco DTP to negotiate the link.

####  Switch S1 Configuration:

```text
S1(config)# interface f0/1
S1(config-if)# switchport mode dynamic desirable
```

**What is happening:** `dynamic desirable` tells S1: "I want to be a trunk! I am going to actively send negotiation packets to the switch across from me on F0/1 to see if we can establish a trunk." Because S2's default configuration on F0/1 is `dynamic auto` (which is willing to become a trunk if asked), the two ports dynamically negotiate, agree, and transition to active 802.1Q trunk ports.

---

### Step 4: Verify DTP Negotiation

 **Verification Command (S1 & S2):**

```text
S1# show interfaces trunk
```

**What is happening:** You will see port Fa0/1 listed with a Mode of `desirable`, Encapsulation as `n-802.1q`, and Status as `trunking`. S1 and S2 can now carry traffic for multiple VLANs simultaneously over this single physical wire.

---

### Step 5: Manually Configure Static Trunks

To ensure stable link connectivity, prevent negotiation loops, and satisfy grading requirements, we statically lock down the trunk configurations between S1 and S3 on both ends.

####  Switch S1 Configuration:

```text
S1(config)# interface f0/3
S1(config-if)# switchport mode trunk
```

####  Switch S3 Configuration:

```text
S3(config)# interface f0/3
S3(config-if)# switchport mode trunk
```

**What is happening:** Hardcoding `switchport mode trunk` on both sides bypasses DTP dynamic negotiation entirely, changing the interface dynamic status to static and locking the link into trunking mode immediately.

---

### Step 6: Creating VLANs on the VTP Server

We create the corporate VLAN segments only on our VTP Server (S2).

####  Switch S2 (Server) Configuration:

```text
S2(config)# vlan 10
S2(config-vlan)# name Red
S2(config-vlan)# exit
S2(config)# vlan 20
S2(config-vlan)# name Blue
S2(config-vlan)# exit
S2(config)# vlan 30
S2(config-vlan)# name Yellow
S2(config-vlan)# exit
S2(config)# vlan 99
S2(config-vlan)# name Management
S2(config-vlan)# end
```

**What is happening:** S2 updates its local database. The database Configuration Revision Number increments. S2 immediately broadcasts a VTP Summary Advertisement out of its trunk interfaces. S1 and S3 receive this, verify the password, notice the higher revision number, and automatically clone the VLANs.

---

### Step 7: VTP Client Block Test

We verify that VTP Clients are locked down as read-only.

####  Switch S1 (Client) Attempted Config:

```text
S1(config)# vlan 10
```

**What is happening (Expected Error):**

```text
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

This is a critical security safeguard. Because S1 is a client, it prevents local administrators from manually tampering with the VLAN database, keeping the VLAN architecture completely unified across the network.

---

### Step 8: Verify Client Database Updates

 **Verification Command (S1 & S3):**

```text
S1# show vlan brief
```

**What is happening:** Even though we never typed `vlan 10`, `vlan 20`, etc., on S1 or S3, they will show Red, Blue, Yellow, and Management as fully active and updated VLANs in their database lists because of VTP.

---

### Step 9: Port-to-VLAN Mapping (Access Ports)

Now that the VLAN databases are in sync, we assign the physical user ports to their respective VLAN broadcast segments.

####  Switch S1 (PC-A assignment):

```text
S1(config)# interface f0/6
S1(config-if)# switchport mode access
S1(config-if)# switchport access vlan 10
```

####  Switch S2 (PC-B assignment):

```text
S2(config)# interface f0/18
S2(config-if)# switchport mode access
S2(config-if)# switchport access vlan 20
```

####  Switch S3 (PC-C assignment):

```text
S3(config)# interface f0/18
S3(config-if)# switchport mode access
S3(config-if)# switchport access vlan 10
```

**What is happening:** Placing the interfaces into access mode restricts them to carrying traffic for only one specific VLAN. PC-A and PC-C are now segmented into VLAN 10 (Red Broadcast Domain), while PC-B is isolated inside VLAN 20 (Blue Broadcast Domain).

---

### Step 10: Configure SVI Management IPs

We establish Switch Virtual Interfaces (SVI) so network engineers can remotely manage the switches over the network.

####  Switch S1 SVI Configuration:

```text
S1(config)# interface vlan 99
S1(config-if)# ip address 192.168.99.1 255.255.255.0
S1(config-if)# no shutdown
```

####  Switch S2 SVI Configuration:

```text
S2(config)# interface vlan 99
S2(config-if)# ip address 192.168.99.2 255.255.255.0
S2(config-if)# no shutdown
```

####  Switch S3 SVI Configuration:

```text
S3(config)# interface vlan 99
S3(config-if)# ip address 192.168.99.3 255.255.255.0
S3(config-if)# no shutdown
```

**What is happening:** SVI provides a virtual host interface on the switch. Entering the `no shutdown` command moves the line protocol state to up, allowing us to ping and configure these switches from any management station inside VLAN 99.

---

### Step 11: End-to-End Connectivity Verification

After statically configuring the IP parameters on PC-A, PC-B, and PC-C, we test the network limits.

**Test 1:** Ping PC-A (`192.168.10.1`) -> PC-B (`192.168.20.1`)

- **Result:** Failed
- **Why:** PC-A belongs to VLAN 10 and PC-B belongs to VLAN 20. Layer 2 switches cannot pass traffic between different VLANs without a Layer 3 Router (Inter-VLAN Routing) configured.

**Test 2:** Ping PC-A (`192.168.10.1`) -> PC-C (`192.168.10.2`)

- **Result:** Successful
- **Why:** Both computers reside within VLAN 10 and can successfully pass ICMP frames across the S1-to-S3 trunk links.

**Test 3:** Ping PC-A (`192.168.10.1`) -> S1 SVI (`192.168.99.1`)

- **Result:** Failed
- **Why:** PC-A is on VLAN 10, whereas S1's management SVI resides in VLAN 99. Since they are on different logical networks, they cannot communicate without routing.

**Test 4:** Ping S1 (`192.168.99.1`) -> S2 (`192.168.99.2`)

- **Result:** Successful
- **Why:** Both switch virtual management interfaces are in the same VLAN (99) and share the same subnet (192.168.99.0/24).

---

### Step 12: Working with Extended VLANs

VTP Versions 1 and 2 restrict databases to Standard-range VLANs (1–1005). If we need to scale out our network up to Extended-range VLANs (1006–4094), we must change the VTP mode on S1 to Transparent Mode.

####  Switch S1 Configuration:

```text
S1(config)# vtp mode transparent
S1(config)# vlan 2000
S1(config-vlan)# end
```
