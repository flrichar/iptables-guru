# Traffic Flow Through the `nat` Table in `kube-proxy`

Reference Document: [Kube-Proxy IPTables Nat Table Control Flow](https://docs.google.com/drawings/d/1MtWL8qRTs6PlnJrW4dh8135_S9e2SaawT410bJuoBPk)  
Man Page Reference: [IPtables Git Repo](https://git.netfilter.org/iptables/)

When `kube-proxy` operates in `iptables` mode, it creates and manages custom chains within the `nat` table to handle virtual IP (VIP) load balancing, packet routing, and network address translation (DNAT/SNAT).

---

## Control Flow Diagram

```
                      [ Local Process / Pod / External Packet ]
                                         │
                   ┌─────────────────────┴─────────────────────┐
                   ▼                                           ▼
              nat.OUTPUT                               nat.PREROUTING
                   │                                           │
                   └─────────────────────┬─────────────────────┘
                                         ▼
                                   KUBE-SERVICES
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        ▼                                ▼                                ▼
  [ Cluster IP ]                  [ External IP ]                   [ LoadBalancer IP ]
        │                                │                                │
        ├─ Masquerade All? ──► MARK      └─ Off-node / Local checks       ▼
        ▼                                                              KUBE-FW-*
    KUBE-SVC-*                                                            │
        │                                                                 ├─ Policy Local? ──► MARK
        │                                                                 ├─ Source Ranges? ──► DROP MARK
        │                                                                 ▼
        │                                                            KUBE-XLB-*
        │                                                                 │
        └────────────────────────────────┬────────────────────────────────┘
                                         ▼
                                     KUBE-SEP-*
                                         │
                                         ├─ Hairpin (Src == Dst IP)? ──► KUBE-MARK-MASQ
                                         ▼
                                  DNAT to Endpoint IP:Port
```

---

## Detailed Step-by-Step Flow

### 1. Entry Chains (`nat.OUTPUT` / `nat.PREROUTING`)
- **Local Host Traffic:** Originating from local processes on the node enters the `nat` table via `nat.OUTPUT`.
- **Pod & Inbound Network Traffic:** Packets arriving from external interfaces or local pod interfaces enter via `nat.PREROUTING`.
- Both built-in chains jump immediately to the `KUBE-SERVICES` chain.

### 2. Service Classification & Matching (`KUBE-SERVICES`)
The `KUBE-SERVICES` chain acts as the primary multiplexer, evaluating packet attributes against defined Kubernetes services:

- **Cluster IP Matches:**
  - Evaluates if `--masquerade-all` is active. If enabled, jumps to `KUBE-MARK-MASQ`.
  - Passes traffic to the service-specific `KUBE-SVC-<HASH>` chain.
- **External IP Matches:**
  - Jumps to `KUBE-MARK-MASQ`.
  - Checks if traffic is coming from off-node and heading to a local IP. If not local, routes packet out to the network.
- **LoadBalancer IP Matches:**
  - Jumps to `KUBE-FW-<HASH>` (Firewall chain).
  - Evaluates `externalTrafficPolicy: Local`. If not local, marks with `KUBE-MARK-MASQ`.
  - Evaluates `loadBalancerSourceRanges`. If the source IP matches, traffic continues; if not matched, traffic jumps to `KUBE-MARK-DROP`.
  - Passes traffic to `KUBE-XLB-<HASH>`.
- **NodePort Matches:**
  - Passes traffic through `KUBE-NODEPORTS`.
  - Evaluates `externalTrafficPolicy: Local`. If local and originating from localhost, marks with `KUBE-MARK-MASQ`.

### 3. Load Balancing & Endpoint Selection (`KUBE-SVC-*` / `KUBE-XLB-*`)
- Evaluates **Session Affinity** using `iptables -m recent`.
- If an existing affinity session matches, traffic bypasses random selection and routes directly to the previously assigned `KUBE-SEP-<HASH>` (Service Endpoint chain).
- If no affinity match exists, `kube-proxy` selects an endpoint using `iptables -m statistic --mode random --probability 1/N` to distribute load equally across available pod endpoints.

### 4. Hairpin NAT & Final DNAT Target (`KUBE-SEP-*`)
- **Hairpin NAT Check:** If the packet source IP is identical to the destination Pod IP (a pod attempting to communicate with its own service VIP), the packet jumps to `KUBE-MARK-MASQ`. This ensures that reply traffic undergoes SNAT and returns properly through the node gateway interface rather than dropping.
- **Connection Tracking Update:** Updates the `recent` module table for session persistence.
- **DNAT Target:** Applies `DNAT --to-destination <pod-ip>:<pod-port>`, rewriting the destination packet header.

### 5. Helper Marking Targets
- `KUBE-MARK-MASQ`: Applies a firewall mark bit (e.g., `0x4000/0x4000`) so that when the packet reaches `nat.POSTROUTING`, it is matched and translated via `MASQUERADE` (SNAT).
- `KUBE-MARK-DROP`: Applies a firewall mark bit (e.g., `0x8000/0x8000`), marking the packet to be rejected/dropped when it transitions into the `filter` table.
