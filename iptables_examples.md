# 11 Practical iptables Examples & Usage Guide

This document contains 11 different `iptables` rules and configuration patterns, including comments and explanations. They demonstrate various tables (`filter`, `nat`, `mangle`), match extensions (`conntrack`, `multiport`, `limit`, `iprange`, `mac`, `time`), and targets (`ACCEPT`, `DROP`, `DNAT`, `MASQUERADE`, `LOG`, `DSCP`).

For complete syntax and option descriptions, refer to the local man page [iptables.8.in](file:///home/flr/Projects/iptables-guru/iptables/iptables/iptables.8.in).

---

### 1. Default Policy Hardening (Default Deny)
Set default chain policies to `DROP` for incoming and forwarded traffic while allowing outgoing connections by default.
```bash
# Change default policy for INPUT and FORWARD chains to DROP (security best practice)
iptables -P INPUT DROP
iptables -P FORWARD DROP

# Allow default outbound traffic from the host
iptables -P OUTPUT ACCEPT
```

---

### 2. Stateful Inspection (Allow Loopback & Established Connections)
Allow internal loopback communication and any incoming packets that belong to existing, established, or related connections.
```bash
# Allow all traffic on local loopback interface (lo)
iptables -A INPUT -i lo -j ACCEPT

# Allow packets for connections already established or related to an active session
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

---

### 3. Open Specific Inbound Ports (SSH, HTTP, HTTPS)
Use the `multiport` match extension to efficiently open multiple TCP ports in a single rule.
```bash
# Allow new incoming TCP connections on SSH (22), HTTP (80), and HTTPS (443)
iptables -A INPUT -p tcp -m multiport --dports 22,80,443 -m conntrack --ctstate NEW -j ACCEPT
```

---

### 4. Destination NAT / Port Forwarding (DNAT)
Forward incoming traffic targeting a specific public port on the gateway to an internal private server IP and port.
```bash
# Redirect incoming TCP port 8080 on WAN interface (eth0) to internal web server 192.168.1.100 on port 80
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80
```

---

### 5. Source NAT / Internet Gateway Masquerading (SNAT / MASQUERADE)
Enable LAN hosts to route to the internet by dynamically rewriting outbound packet source IP addresses to match the gateway's public interface.
```bash
# Translate source IP of outbound traffic leaving WAN interface (eth0) to the interface's dynamic IP
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

### 6. Rate Limiting ICMP (Ping Flood Mitigation)
Limit incoming `ICMP echo-request` packets to prevent denial-of-service (DoS) or ping flood attacks.
```bash
# Accept ICMP echo requests up to 1 per second, allowing an initial burst of 5 packets
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s --limit-burst 5 -j ACCEPT

# Drop any ICMP echo requests exceeding the rate limit
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

---

### 7. Blocking IP Ranges (`iprange` match)
Block network traffic originating from a contiguous block of IP addresses.
```bash
# Drop all incoming packets from source IP range 192.168.1.100 through 192.168.1.150
iptables -A INPUT -m iprange --src-range 192.168.1.100-192.168.1.150 -j DROP
```

---

### 8. Logging Suspicious or Dropped Packets (`LOG` target)
Log attempts to access unauthorized ports (such as Telnet) to `dmesg`/`syslog` before dropping the packet.
```bash
# Log Telnet access attempts (TCP 23) with custom prefix, rate-limited to avoid log flooding
iptables -A INPUT -p tcp --dport 23 -m limit --limit 5/m -j LOG --log-prefix "IPTABLES-TELNET-ATTEMPT: " --log-level 4

# Drop the Telnet connection attempts
iptables -A INPUT -p tcp --dport 23 -j DROP
```

---

### 9. MAC Address Filtering (`mac` match)
Restrict network ingress based on the hardware MAC address of the connecting network card.
```bash
# Explicitly allow incoming traffic from specific MAC address on eth0
iptables -A INPUT -i eth0 -m mac --mac-source 00:11:22:33:44:55 -j ACCEPT
```

---

### 10. Time-Based Traffic Blocking (`time` match)
Block outgoing HTTP web traffic during work hours (Monday through Friday, 09:00 to 17:00 UTC).
```bash
# Block outgoing HTTP traffic (port 80) during weekdays from 09:00 to 17:00 UTC
iptables -A OUTPUT -p tcp --dport 80 -m time --timestart 09:00 --timestop 17:00 --weekdays Mon,Tue,Wed,Thu,Fri -j DROP
```

---

### 11. Quality of Service Packet Marking (`mangle` table & `DSCP`)
Modify the Differentiated Services Code Point (DSCP) field in packet headers to prioritize SSH traffic for Quality of Service (QoS) routing.
```bash
# Set DSCP class Expedited Forwarding (EF / high priority) for outbound SSH traffic in mangle table
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j DSCP --set-dscp-class EF
```
