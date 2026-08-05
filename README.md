# IPTables, the Good, the Bad, & the Ugly

What is IPTables? It's the kernel's user-space tool companion for Netfilter, the main packet filtering subsystem inside the Linux Kernel.

It consists of Tables, that host Chains, and are comprised of Rules. The primary tables are:
 * Filter (default)
 * Nat
 * Mangle

There are also `raw` and `security` tables, but used less often.  

Each table has a set of chains, and those chains are either built-in or custom. The primary packet filtering mechanism is handled by `filter`. Nat handles masquerading/nat which alters the source and desintation of packets.  Mangle is for manipulating the IP packet headers.

The built-in chains are listed below:

```
PREROUTING
    This chain is applied to all incoming packets. 

INPUT
    This chain is applied to packets destined for the system's internal processes. 

FORWARD
    This chain is applied to packets that are only routed through the system. 

OUTPUT
    This chain is applied to packets originating from the system itself. 

POSTROUTING
    This chain is applied to all outgoing packets. 
```
...

## Filter Chains
* INPUT, FORWARD, OUTPUT 

## Nat Chains
* PREROUTING, INPUT, OUTPUT, POSTROUTING

## Mangle Chains
* PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING

The SUSE Docs have a great example of how these map and the packet's path through the host.
* [SUSE SLE Masquerading & Firewalls](https://documentation.suse.com/sles/15-SP4/html/SLES-all/cha-security-firewall.html)

Chains not listed as built-in are Custom Chains, defined by some outside process. Each chain has a series of rules that are processed in order, so rule ordering matters. Rules read like a sentence, with flags that outline matching definitions for packets, like in/out interfaces, sources, destinations, and other fields. The best source of `iptables` documentation is the official iptables repo, and the man pages are very well written and thorough.
* [IPtables Git Repo](https://git.netfilter.org/iptables/)

(see various iptables rules examples)

## Kubernetes CNI

Proper operation of the major K8s CNI rely heavily on IPTables. Kube-Proxy is responsible for managing iptables rules for services, where the CNI stages the initial configuration. Kubernetes does prefix all rules with `KUBE-` and inlcudes comments like `/* example comment with rule description or other important info */`.
(See diagrams for kube-proxy nat/masquerade flow).

## Bad & Ugly

* Rules are *NOT* aware of each other. One app or security agent can insert or append lines that have a larger effect on the entire chain. This is why disabling firewalld / external firewall rules is important. Firewalld might work with a zone-based firewall, recommended only in a lab or dev setting, trusting the kubernetes zone or segment.
* The userspace `iptables` kernel interface has evolved over the years, the `legacy` mode is deprecated and should move onto `nftables`. *Using both on the same node can cause unintended operations!*
* eBPF changes the calculus of the rules, because it is bytecode that can be inserted directly into the kernel for networking, security, and observability, but still may require iptables rules for proper Kubernetes CNI operation.

## Wait, an Agent?

The `iptables` dir locally would just include a copy of the netfilter official `iptables` repo, pull it as a submodule as below (it's ignored via .gitignore).

* `git submodule add https://git.netfilter.org/iptables`

---
