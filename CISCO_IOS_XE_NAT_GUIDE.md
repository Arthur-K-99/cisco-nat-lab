# 📘 Comprehensive Guide to Cisco IOS-XE NAT

This guide serves as a complete reference for all Network Address Translation (NAT) scenarios on Cisco IOS-XE platforms. It details the underlying architecture, order of operations, exact CLI syntax, packet flow behaviors, and advanced troubleshooting techniques.

---

## 1. Cisco IOS-XE NAT Architecture

Network Address Translation on Cisco IOS-XE differs fundamentally from legacy IOS due to the separation of the Control Plane (running IOS-d) and the Data Plane (handled by the Quantum Flow Processor - QFP). NAT is executed in the data plane using flow-based processing.

There are two primary styles of NAT configuration in IOS-XE:

### A. Classic NAT (Interface-Based)
Classic NAT uses directional interface roles:
- **`ip nat inside`**: Identifies interfaces facing the private network.
- **`ip nat outside`**: Identifies interfaces facing the public network/internet.

#### Classic NAT Order of Operations
Understanding the order of operations is critical. It determines whether routing occurs before translation:

```
  ┌─────────────────────────────────────────────────────────┐
  │              INSIDE → OUTSIDE Traffic Flow               │
  │                                                         │
  │  1. Routing decision (uses INSIDE LOCAL address)        │
  │  2. Check ip nat inside source rules                    │
  │  3. Translate source address                            │
  │  4. Forward packet out outside interface                │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │              OUTSIDE → INSIDE Traffic Flow               │
  │                                                         │
  │  1. Translate destination (INSIDE GLOBAL → LOCAL)       │
  │  2. Routing decision (uses INSIDE LOCAL address)        │
  │  3. Forward packet out inside interface                 │
  └─────────────────────────────────────────────────────────┘
```
> [!IMPORTANT]
> Because routing occurs **before** NAT for inside-to-outside traffic, the router must already have a route to the destination using the *pre-NAT* source IP.
> Conversely, for outside-to-inside traffic, NAT occurs **before** routing, meaning the router routes using the *post-NAT* destination IP.

---

### B. NAT Virtual Interface (NVI / Domainless NAT)
NVI removes the concept of strict inside/outside interfaces:
- Interfaces are configured with **`ip nat enable`**.
- The router dynamically builds NVI interfaces (`nvi0`) to handle routing.
- Direct inside-to-inside hairpinning is supported natively without reflecting traffic to an ISP.

---

## 2. Cisco IOS-XE NAT Scenario Directory

This directory details the configuration, use-cases, and flows for the nine primary NAT types.

### scenario ①: Static NAT (1:1 Inside Source)
Maps a single private IP address to a single public IP address permanently and bidirectionally.
- **Use-Case**: Internal servers (e.g., SMTP or database) that must be directly reachable from the outside.
- **CLI Syntax**:
  ```text
  ip nat inside source static <inside-local-ip> <inside-global-ip> [extendable]
  ```
- **Packet Flow**:
  - **Inbound**: Destination is translated from global (public) to local (private).
  - **Outbound**: Source is translated from local (private) to global (public).

---

### scenario ②: Static PAT (Port Forwarding)
Maps a specific port on an inside local address to a specific port on an inside global address. This allows multiple internal servers to share a single public IP.
- **Use-Case**: Hosting a web server (port 80) and mail server (port 25) on different local hosts using the same WAN IP.
- **CLI Syntax**:
  ```text
  ip nat inside source static {tcp | udp} <inside-local-ip> <local-port> <inside-global-ip> <global-port> [extendable]
  ```
- **Packet Flow**:
  - **Inbound**: `dst_ip:port` is translated to the internal private IP and port.
  - **Outbound**: `src_ip:port` is translated to the public WAN IP and global port.

---

### scenario ③: Dynamic NAT Pool (Many-to-Many)
Dynamically maps inside local addresses to a pool of public IP addresses on a first-come, first-served basis. No port multiplexing occurs.
- **Use-Case**: Legacy applications that do not support port multiplexing (PAT) but still require dynamic IP allocation.
- **CLI Syntax**:
  ```text
  ip nat pool <POOL-NAME> <start-ip> <end-ip> netmask <mask>
  ip access-list extended <ACL-NAME>
    permit ip <local-subnet> <wildcard> any
  ip nat inside source list <ACL-NAME> pool <POOL-NAME>
  ```
- **Gotcha**: Pool exhaustion will cause new connection requests to be dropped. Monitor with `show ip nat statistics`.

---

### scenario ④: Dynamic PAT / Overload (Many-to-One / Many-to-Few)
Maps multiple inside local addresses to a single public IP address (or small pool) by multiplexing source ports (up to ~65,000 concurrent flows per IP).
- **Use-Case**: Standard internet access for office LANs.
- **CLI Syntax**:
  - **Interface-based PAT**:
    ```text
    ip nat inside source list <ACL-NAME> interface <interface-id> overload
    ```
  - **Pool-based PAT**:
    ```text
    ip nat inside source list <ACL-NAME> pool <POOL-NAME> overload
    ```

---

### scenario ⑤: Policy NAT (Selective Translation)
Applies NAT rules selectively based on traffic characteristics, such as destination subnets or protocols.
- **Use-Case**: Directing traffic going to a partner network to appear from a specific business IP, while general internet traffic uses standard PAT.
- **CLI Syntax (Route-Map Based)**:
  ```text
  ip access-list extended POLICY-ACL
    permit ip 192.168.20.0 0.0.0.255 203.0.113.0 0.0.0.255
  route-map POLICY-MAP permit 10
    match ip address POLICY-ACL
  !
  ip nat inside source route-map POLICY-MAP pool <POOL-NAME> [overload]
  ```

---

### scenario ⑥: Outside Source NAT
Translates the source IP of packets arriving from the outside network. Used to represent outside hosts as if they belong to a local subnet.
- **Use-Case**: Resolving routing conflicts when connecting to external networks that overlap with local address ranges.
- **CLI Syntax**:
  ```text
  ip nat outside source static <outside-global-ip> <outside-local-ip>
  ```
- **Terminology**:
  - **Outside Global**: The real IP address of the host in the outside network.
  - **Outside Local**: The IP address that the inside network perceives the outside host to have.

---

### scenario ⑦: Twice NAT (Overlapping Address Spaces)
Translates both the source and the destination addresses of a packet simultaneously.
- **Use-Case**: Resolving IP address overlaps during company mergers, where both LANs use the exact same private subnet (e.g., `192.168.10.0/24`).
- **CLI Configuration Example**:
  ```text
  ! BR2 Router configuration
  ! Step 1: Translate local source
  ip nat inside source static 192.168.10.10 10.10.10.10
  ! Step 2: Translate remote destination
  ip nat outside source static 192.168.10.10 10.20.20.10
  ! Step 3: Route the virtual destination subnet to the WAN
  ip route 10.20.20.0 255.255.255.0 100.64.0.5
  ```

---

### scenario ⑧: NAT Hairpinning (NAT Loopback)
Allows internal clients on an inside interface to access an internal server (also on an inside interface) using the server's public IP address.
- **Classic NAT Implementation (ISP Reflection)**:
  - Inside local client traffic is routed out the WAN interface toward the ISP, then reflected back to the WAN interface to trigger destination translation.
- **NVI Implementation (Direct Hairpin)**:
  - If using `ip nat enable`, the translation occurs dynamically inside the router, keeping the packet path local.

---

### scenario ⑨: VRF-Aware NAT
Translates addresses within specific Virtual Routing and Forwarding (VRF) tables, allowing VRFs to share a public global interface.
- **Use-Case**: Service provider architectures or multi-tenant corporate networks.
- **CLI Syntax**:
  ```text
  ip nat inside source list <ACL> pool <POOL> vrf <VRF-NAME> [match-in-vrf] [overload]
  ```

---

## 3. Advanced Troubleshooting and Debugging

### Essential Verification Commands

| Command | Purpose |
|---------|---------|
| `show ip nat translations` | Displays the active NAT translation table. |
| `show ip nat translations verbose` | Shows translation flags, precise timeouts, and creation times. |
| `show ip nat statistics` | Shows hits/misses, pool limits, allocation counts, and interface mapping. |
| `show access-lists` | Verifies packet match counts on NAT ACLs. |

### Diagnostic CLI Outputs

#### 1. Interpreting `debug ip nat`
```text
NAT*: s=192.168.10.10->198.51.100.16, d=203.0.113.100 [5893]
```
- **`s=192.168.10.10->198.51.100.16`**: Inside local source address `192.168.10.10` translated to inside global IP `198.51.100.16`.
- **`d=203.0.113.100`**: Destination address.
- **`[5893]`**: IP Packet identification number.

#### 2. Clearing NAT Translations safely
> [!WARNING]
> Clearing the NAT translation table will disrupt active TCP and UDP connections. Avoid running wildcard clears in production.
- **Clear specific translation**:
  ```text
  clear ip nat translation inside <local-ip> [global-ip]
  ```
- **Clear all dynamic translations**:
  ```text
  clear ip nat translation *
  ```
- **Force clear stuck translations**:
  ```text
  clear ip nat translation forced
  ```

---

## 4. Key Configuration Keywords Explained

### `extendable`
Allows mapping of the same inside local IP address to multiple inside global IP addresses or ports. It is automatically appended by modern IOS versions but should be typed explicitly when configuring multiple static translations for a single host.

### `no-alias`
Prevents the router from creating an Alias IP address for the translated global address. By default, the router will respond to ARP requests for the inside global IP address. In scenarios where a firewall or load-balancer handles ARP for those IPs, `no-alias` is required to prevent IP conflicts.
