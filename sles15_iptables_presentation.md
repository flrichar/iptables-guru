# Presentation Outline: Linux Packet Filtering & Enterprise Firewalls on SLES 15 SP4

Reference Document: [SUSE SLE Masquerading & Firewalls](https://documentation.suse.com/sles/15-SP4/html/SLES-all/cha-security-firewall.html)  
Man Page Reference: [IPtables Git Repo](https://git.netfilter.org/iptables/)

**Target Presentation Duration:** ~40 Minutes  
**Structure:** 4 Core Modules (~8-10 minutes each) + Q&A

---

## Module 1: Kernel Netfilter Architecture & Packet Flow (10 mins)

### Slide Topics & Talking Points:
1. **Netfilter vs. iptables Userspace:**
   - Understanding Netfilter hooks inside the Linux kernel vs. `iptables` CLI rule definition.
2. **The 3 Core Packet Filtering Tables:**
   - `filter`: Core filtering decisions (`ACCEPT`, `DROP`, `REJECT`).
   - `nat`: Network address translation (`DNAT`, `SNAT`, `MASQUERADE`).
   - `mangle`: IP header modification (Type of Service / DSCP, TTL adjustment).
3. **The 5 Predefined Netfilter Chains:**
   - `PREROUTING`: Incoming packets before routing decisions.
   - `INPUT`: Packets destined for local system sockets.
   - `FORWARD`: Packets routed through the host between interfaces.
   - `OUTPUT`: Packets generated locally by system processes.
   - `POSTROUTING`: Outgoing packets after routing decisions.
4. **Gateway Configuration & Masquerading Basics:**
   - Enforcing system routing: `sysctl net.ipv4.ip_forward = 1` (`/etc/sysctl.conf` / YaST).
   - Dynamic IP translation for LAN internet access using `MASQUERADE`.

---

## Module 2: Enterprise Firewall Management with `firewalld` in SLES 15 (10 mins)

### Slide Topics & Talking Points:
1. **Architectural Transition in SLES 15 GA:**
   - Deprecation of `SuSEfirewall2` and adoption of `firewalld` as the default firewall daemon.
2. **Zone-Based Trust Model:**
   - Network interfaces assigned to trust zones (`public`, `internal`, `dmz`, `trusted`).
   - Querying interface zones: `firewall-cmd --get-default-zone`, `--get-zone-of-interface=eth0`.
3. **Managing Ports and Services via CLI (`firewall-cmd`):**
   - Opening services: `firewall-cmd --add-service=http --zone=internal`.
   - Opening ports: `firewall-cmd --add-port=8000/tcp --zone=internal`.
   - Temporary testing: Utilizing `--timeout=5m` to prevent accidental lockdown.
4. **Runtime vs. Permanent Configuration:**
   - Understanding runtime memory state vs. disk state.
   - Persisting changes: `firewall-cmd --runtime-to-permanent`.
   - Reloading saved state: `firewall-cmd --reload` or `systemctl reload firewalld`.

---

## Module 3: Dynamic RPC Services & Custom Rule Integration (10 mins)

### Slide Topics & Talking Points:
1. **Handling Dynamic Port Services (NFSv3 & NIS):**
   - Challenges introduced by dynamic `rpcbind` port allocation.
   - Pinning static ports via `/etc/sysconfig/nfs` (`MOUNTD_PORT`, `STATD_PORT`) and `/etc/modprobe.d/60-nfs.conf`.
   - Creating custom `firewalld` service XML definitions (`firewall-cmd --new-service=nfs-rpc`).
   - Utilizing the SUSE helper utility: `firewall-rpcbind-helper.py`.
2. **Security Controls & Custom Rule Formats:**
   - **Lockdown Mode:** Preventing unauthorized D-Bus firewall updates (`firewall-cmd --lockdown-on`).
   - **Direct Rules (`--direct`):** Passing raw `iptables` syntax for complex legacy setups.
   - **Rich Rules (`--add-rich-rule`):** Expressive high-level syntax for granular IP/subnet dropping.

---

## Module 4: Legacy Migration & The Modern `nftables` Backend (10 mins)

### Slide Topics & Talking Points:
1. **Migrating from SuSEfirewall2:**
   - Automated conversion tool: `susefirewall2-to-firewalld`.
   - Performing dry-run audits (`-v`) and committing changes (`-c`).
2. **Evolution to `nftables` in SLES 15:**
   - Overview of `nftables` replacing `iptables`, `ip6tables`, `arptables`, `ebtables`, and `ipset`.
   - Unified dual-stack IPv4/IPv6 table structures (`/etc/nftables.conf`).
   - **Technical Advantages:**
     - Atomic ruleset commits (no full table replacement during updates).
     - Modern VM bytecode evaluation.
     - Live debugging and tracing via `nftrace` and `nft`.

---

## Q&A Session (5-10 mins)
- Recommended discussion topics: `kube-proxy` integration with `firewalld`, transitioning direct `iptables` scripts to `nftables`, and verifying live packet flow.
