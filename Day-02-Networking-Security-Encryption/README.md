# Day 2 — Networking & Security Basics

**Phase 1: Foundations & Linux Mastery** · Day 2 of 20

| | |
|---|---|
| **Lectures** | `Day 2_Networking` (22 slides), `Day 7_Intro Security and Encryption` (20 slides) |
| **Lab host** | Ubuntu 24.04 LTS on AWS EC2 — `m7i-flex.large`, 2 vCPU / 8 GiB, `us-east-1a` |
| **Running stack** | Wazuh SIEM + full ELK (7 services) |
| **Access** | SSH via MobaXterm |

Both lectures were theory-only — no commands, no demos. So every claim in both decks was verified against a live server. Every output below is real output from my own instance.

> **Note on redaction:** AWS account ID, instance ID, and the public IP are partially masked below. Private `10.x` addresses are RFC 1918 and safe to publish. Deciding what's safe to put in a public repo is itself part of the job.

---

## Contents

1. [Interfaces and IP addresses](#1--interfaces-and-ip-addresses)
2. [Public vs private IP](#2--public-vs-private-ip)
3. [Routing and the default gateway](#3--routing-and-the-default-gateway)
4. [ping, ICMP and TTL](#4--ping-icmp-and-ttl)
5. [DNS resolution](#5--dns-resolution)
6. [The DNS delegation chain](#6--the-dns-delegation-chain)
7. [Ports and listening services](#7--ports-and-listening-services)
8. [Firewall testing: refused vs timed out](#8--firewall-testing-refused-vs-timed-out)
9. [HTTP vs HTTPS](#9--http-vs-https)
10. [Reading a TLS certificate](#10--reading-a-tls-certificate)
11. [The three encryption types](#11--the-three-encryption-types)
12. [IAM roles, authentication vs authorization](#12--iam-roles-authentication-vs-authorization)
13. [Troubleshooting ladder](#13--troubleshooting-ladder)
14. [Findings](#14--findings)
15. [Key takeaways](#15--key-takeaways)

---

## 1 · Interfaces and IP addresses

*Covers networking slides 10 (IP address), 11 (IPv4 vs IPv6), 18 (commands)*

```bash
ifconfig                  # what the lecture taught
ip -brief addr show       # the modern equivalent
ip addr show              # full detail
```

**Output:**

```
enp39s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 10.20.1.77  netmask 255.255.255.0  broadcast 10.20.1.255
        inet6 fe80::ba:71ff:fecd:a5ef  prefixlen 64  scopeid 0x20<link>
        ether 02:ba:71:cd:a5:ef  txqueuelen 1000  (Ethernet)

lo:     inet 127.0.0.1  netmask 255.0.0.0
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp39s0          UP             10.20.1.77/24 metric 100 fe80::ba:71ff:fecd:a5ef/64
```

| Field | Value | Meaning |
|---|---|---|
| Interface | `enp39s0` | Not `eth0` — modern Linux uses predictable names derived from the PCI slot |
| Private IPv4 | `10.20.1.77/24` | The only IPv4 address the OS has |
| MAC | `02:ba:71:...` | Layer-2 address. Leading `02` = locally administered, i.e. AWS assigned it virtually |
| MTU | `9001` | Jumbo frames. AWS VPCs allow 9001 bytes vs the internet's usual 1500 |
| Loopback | `127.0.0.1/8` | `scope host` — never leaves the machine |
| IPv6 | `fe80::...` | **Link-local only.** Not internet-routable |

**Notes**

- `ifconfig` ran fine here — `net-tools` was installed. It is deprecated and absent on many Ubuntu 24.04 images, so `ip` is the command to build the habit on.
- `/24` = 256 addresses, but **AWS reserves 5 per subnet** (`.0` network, `.1` gateway, `.2` DNS, `.3` reserved, `.255` broadcast) → 251 usable.
- `valid_lft 3348sec` — the address is a DHCP lease with ~56 min remaining. AWS auto-renews; the private IP stays fixed for the instance's life.
- Hostname `ip-10-20-1-77` is derived from the private IP. The shell prompt tells you the IP before you run anything.
- `lo` showing state `UNKNOWN` is normal for loopback, not a fault.

**The observation that matters:** the public IP appears nowhere in this output.

---

## 2 · Public vs private IP

*Covers networking slide 16*

```bash
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type
curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/network/interfaces/macs/<MAC>/vpc-ipv4-cidr-block
```

**Output:**

```
private:  10.20.1.77
public :  3.238.141.xxx
type   :  m7i-flex.large
az     :  us-east-1a
vpc-cidr: 10.20.0.0/16
```

### Network layout

```
Internet
   │
   ▼  3.238.141.xxx   ← public IP, lives on the AWS internet gateway (1:1 NAT)
┌─────────────────────────────────────────┐
│ VPC  10.20.0.0/16       65,536 addrs    │
│  ┌────────────────────────────────────┐ │
│  │ Subnet 10.20.1.0/24   251 usable   │ │
│  │   enp39s0 → 10.20.1.77             │ │
│  │   ← the only address the OS sees    │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
       us-east-1a  ·  m7i-flex.large
```

**Why `ip addr` and "what's my IP" disagree:** the public IP is never configured on the interface. AWS's internet gateway performs 1:1 NAT between `3.238.141.xxx` and `10.20.1.77`. The OS only ever sees the private side. This is slide 16's public/private distinction as an observable fact rather than a definition.

**Two things this establishes:**

- The VPC is a `/16`, so it has room for 256 subnets like this one. One is in use.
- **The DNS resolver will be at `10.20.0.2`** — AWS always places it at VPC base + 2. Confirmed in §5.

**Operational warning:** unless it's an Elastic IP, `3.238.141.xxx` changes on stop/start. Anything hardcoding it — notes, SG rules, bookmarks — breaks.

**`169.254.169.254` is link-local.** Not routable, identical on every EC2 instance, only answers from inside. The `PUT`-for-a-token requirement is IMDSv2 — see §12 for why that matters enormously.

---

## 3 · Routing and the default gateway

*Covers networking slide 7 (What is a Router?)*

```bash
ip route show
ip route get 8.8.8.8        # internet destination
ip route get 10.20.1.50     # same-subnet destination
```

**Output:**

```
default      via 10.20.1.1 dev enp39s0 proto dhcp src 10.20.1.77 metric 100
10.20.0.2    via 10.20.1.1 dev enp39s0 proto dhcp src 10.20.1.77 metric 100
10.20.1.0/24 dev enp39s0 proto kernel scope link src 10.20.1.77 metric 100
10.20.1.1    dev enp39s0 proto dhcp scope link src 10.20.1.77 metric 100
```

```
8.8.8.8 via 10.20.1.1 dev enp39s0 src 10.20.1.77     ← has "via"
10.20.1.50 dev enp39s0 src 10.20.1.77                 ← no "via"
```

### The word `via` is the whole lesson

**`via` means "hand this to a router."** No `via` means "the destination is on my own wire — I'll deliver it myself."

For `10.20.1.50`, the instance ARPs ("who has 10.20.1.50?"), gets a MAC back, and sends the frame directly. For `8.8.8.8` it can't — that address isn't local — so it sends the packet to `10.20.1.1`'s MAC and lets the router take over. That is a router's entire job, visible in one word.

### Reading the table

| Route | Meaning |
|---|---|
| `default` | Shorthand for `0.0.0.0/0` — the catch-all, last resort |
| `10.20.0.2 via ...` | The DNS resolver is in the VPC but **not** in this `/24`, so it needs the gateway |
| `10.20.1.0/24 scope link` | My own subnet — direct delivery |
| `10.20.1.1 scope link` | The gateway itself |

- **Routes match longest-prefix-first, not top-to-bottom.** `10.20.1.50` matches both `default` and `10.20.1.0/24`; the more specific `/24` wins. This is why local traffic never accidentally exits via the gateway.
- **`proto dhcp` means none of this was configured by hand.** AWS's DHCP server pushed these routes at boot.
- **`10.20.1.1` is not an appliance.** It's the VPC's implicit router — a distributed AWS software function present at subnet base + 1 in every subnet. There is nothing to log into.

---

## 4 · ping, ICMP and TTL

*Covers networking slide 18*

```bash
ping -c 3 10.20.1.1      # the gateway
ping -c 3 8.8.8.8        # internet by IP
ping -c 3 google.com     # internet by name
```

**Output (abridged):**

```
64 bytes from 10.20.1.1: icmp_seq=1 ttl=64  time=0.070 ms
rtt min/avg/max/mdev = 0.070/0.197/0.310/0.098 ms

64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1.95 ms
rtt min/avg/max/mdev = 1.951/2.047/2.221/0.123 ms

PING google.com (142.251.111.139)
64 bytes from bk-in-f139.1e100.net (142.251.111.139): ttl=102 time=2.40 ms
rtt min/avg/max/mdev = 2.403/2.431/2.447/0.020 ms
```

All three succeeded — the AWS VPC router **does** answer ICMP from inside the VPC.

### TTL is a free distance measurement

TTL starts at a fixed value on the sender and **decrements by 1 at every router crossed**. Its actual purpose is to stop packets looping forever, but the received value tells you roughly how far away the responder is.

| Target | TTL | Interpretation |
|---|---|---|
| `10.20.1.1` | 64 | Nothing decremented it — directly attached, zero routers between |
| `8.8.8.8` | 117 | ~11 hops (assuming a 128 start) |
| `google.com` | 102 | ~26 hops |

Rough estimate only — the initial value varies by OS (64, 128, or 255) — but it's free information on every ping.

### Latency

| Target | Avg | Why |
|---|---|---|
| `10.20.1.1` | 0.197 ms | Same hypervisor fabric, never leaves AWS's local network |
| `8.8.8.8` | 2.047 ms | us-east-1 peers directly with Google |
| `google.com` | 2.431 ms | Same |

2 ms to Google is fast — a home connection typically sees 10–30 ms.

Note the gateway's spread: min `0.070`, max `0.310` — a 4× swing. The local link has higher *relative* jitter than the internet path because it's software-emulated rather than a physical wire. `mdev` is the column that reveals this.

### `ping <hostname>` tests three things at once

1. **DNS worked** — it printed `142.251.111.139`
2. **Reverse DNS worked** — it also printed `bk-in-f139.1e100.net`, so it looked the IP back up to a name. `1e100.net` is Google's infrastructure domain (`1e100` = 1 googol)
3. **Routing and return path worked**

This is why `ping <hostname>` beats `ping <ip>` as a first test. If it works, a lot is working. If it fails, the next command is `ping 8.8.8.8` — success there narrows the fault to **DNS alone**.

### A stateful firewall, demonstrated accidentally

The Security Group has no inbound ICMP rule — only SSH on 22. The ping replies came back anyway.

That is statefulness (security slide 6). The SG remembered that *this instance* initiated the outbound echo request and permitted the matching reply automatically. A stateless firewall — a Network ACL — would have dropped every reply until an explicit inbound rule was added. See §8 and §14.

---

## 5 · DNS resolution

*Covers networking slide 12. (Note: the slide says "Domain Name **Server**" — the correct expansion is Domain Name **System**.)*

```bash
sudo apt install -y dnsutils
dig google.com
cat /etc/resolv.conf
resolvectl status
```

**Output:**

```
google.com.  119  IN  A  142.251.111.101
google.com.  119  IN  A  142.251.111.139
;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

```
nameserver 127.0.0.53
options edns0 trust-ad
search ec2.internal
```

```
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub
Link 2 (enp39s0)
Current DNS Server: 10.20.0.2
       DNS Servers: 10.20.0.2
        DNS Domain: ec2.internal
```

### There are two resolvers, not one

```
dig  →  127.0.0.53          →  10.20.0.2            →  public DNS hierarchy
        systemd-resolved       AWS Route 53 Resolver     root → com → google.com
        local stub, caching    (VPC base + 2)            (authoritative)
```

`127.0.0.53` is a caching stub resolver running **on the instance itself**. Every DNS query on Ubuntu 24.04 hits it first; anything uncached is forwarded upstream. `resolv.conf mode: stub` names this arrangement, and `resolvectl status` confirms the real upstream is `10.20.0.2` — exactly as predicted from the VPC CIDR in §2.

Corroborating evidence from §3: the routing table carries a dedicated route for `10.20.0.2`. Nothing would push a specific route to an address that's never used.

### Caching, caught in the act

```bash
dig google.com | grep -E "^google.com|Query time"
sleep 5
dig google.com | grep -E "^google.com|Query time"
```

```
google.com.  256  IN  A  142.251.111.102 / .139 / .138 / .113 / .100 / .101
;; Query time: 1 msec
                     ↓  sleep 5
google.com.  250  IN  A  142.251.111.113 / .102 / .100 / .139 / .101 / .138
;; Query time: 0 msec
```

**256 → 250.** The TTL decremented in real time. Nothing re-fetched — the stub returned the same cached entry with less life left.

Compare against the first `dig`: TTL 119 with 2 records, then TTL 256 with 6. **The TTL went up**, which is impossible for one cached entry — so it expired and re-fetched between commands, returning a different, larger address set.

And the ordering rotated between the two runs. Same six addresses, different order. That's **DNS round-robin** — clients use the first address returned, so shuffling spreads load across Google's front-ends. Load balancing, visible in a text field.

**This mechanism is why "DNS changes take time to propagate":** every cache between you and the authoritative server holds the old answer until its TTL expires. It's also why you lower TTLs *before* a planned migration.

### `Query time` interpreted correctly

| Resolver | Query time |
|---|---|
| local stub `127.0.0.53` | 0–1 ms |
| AWS resolver `10.20.0.2` | 0 ms |
| `8.8.8.8` (wikipedia.org) | 6 ms |
| `8.8.8.8` (archive.org) | 3 ms |

`dig` reports whole milliseconds, so **`0 msec` means "under half a millisecond," not necessarily "cached."** The query to `10.20.0.2` genuinely left the instance — it just returned sub-millisecond, since the gateway pings at 0.197 ms and AWS's resolver keeps its own cache. Queries that truly cross a network cost measurable time, as the `8.8.8.8` rows show.

### Two more details

- **`search ec2.internal`** — an AWS-injected search domain. An unqualified `ping myserver` silently also tries `myserver.ec2.internal`. Useful inside a VPC, confusing when you don't know it's happening.
- **`/etc/resolv.conf` says `# Do not edit`** — it's a symlink to a file `systemd-resolved` regenerates. Editing it appears to work and silently reverts on reboot.

### Two security findings in that output

```
DNSSEC=no/unsupported
-DNSOverTLS
```

- **No DNSSEC** — the resolver does not cryptographically verify that answers are genuine. A spoofed reply would be accepted.
- **No DNS-over-TLS** (`-` prefix = disabled) — DNS queries travel in **plaintext**. Every domain looked up is visible to anything on the path.

Worth sitting with: HTTPS encrypts your traffic, but the DNS lookup that happens *first* — the one revealing which site you're visiting — is not encrypted here. **Encryption at one layer does not protect the layer below it.**

---

## 6 · The DNS delegation chain

```bash
dig +trace google.com
```

**Output (abridged):**

```
.            86397   IN  NS  a–m.root-servers.net.          (13 servers)
;; Received 239 bytes from 127.0.0.53#53 in 2 ms

com.         172800  IN  NS  a–m.gtld-servers.net.          (13 servers)
com.         86400   IN  DS     19718 13 2 8ACBB0CD...
com.         86400   IN  RRSIG  DS 8 1 86400 ...
;; Received 1198 bytes from 192.33.4.12#53(c.root-servers.net) in 2 ms

google.com.  172800  IN  NS  ns1–ns4.google.com.            (4 servers)
CK0POJMG...com. 900 IN NSEC3 1 1 0 - ... NS SOA RRSIG DNSKEY NSEC3PARAM
;; Received 644 bytes from 192.26.92.30#53(c.gtld-servers.net) in 2 ms

google.com.  300     IN  A   192.178.155.138 / .139 / .102 / .113 / .100 / .101
;; Received 135 bytes from 216.239.32.10#53(ns1.google.com) in 2 ms
```

### The hierarchy

| Stage | Servers | TTL | Meaning |
|---|---|---|---|
| `.` root | 13 (`a`–`m.root-servers.net`) | 86397 | ~24 h — root hints almost never change |
| `com.` | 13 (`a`–`m.gtld-servers.net`) | 172800 | 48 h — TLD servers are extremely stable |
| `google.com.` | 4 (`ns1`–`ns4.google.com`) | 172800 | 48 h |
| **Answer** | 6 A records | **300** | 5 min — Google wants agility |

Four round trips, 2 ms each. **No single server in that chain knew the final answer** — it was assembled by walking a hierarchy.

**Why exactly 13 roots?** Original DNS replies had to fit in a 512-byte UDP packet, and 13 nameserver records was the maximum that fit. The limit stuck. Those 13 *names* are served by hundreds of physical machines worldwide via **anycast** — which is why `c.root-servers.net` answered in 2 ms rather than from wherever it nominally lives.

**The TTL story completes here.** The authoritative server publishes **300**. The 256, 250 and 119 seen in §5 were all that same 300 counting down in different caches at different moments.

### GeoDNS: same name, different answers, both correct

```
via local resolver:   142.251.111.x
via +trace:           192.178.155.x
```

`+trace` bypassed the local resolver and queried `ns1.google.com` directly from the instance. Google inspects **who is asking** and returns whichever front-end suits that asker. When AWS's resolver asks, one set; when this box asks directly, another.

Operational consequence: **"which IP does this domain resolve to" has no single answer.** It depends on which resolver you ask and where it sits. Teams lose hours when a colleague can't reproduce a DNS result.

### DNSSEC: the crypto is published, the resolver ignores it

```
com.  DS     19718 13 2 8ACBB0CD...    ← hash of com's signing key
com.  RRSIG  DS 8 1 86400 ...          ← cryptographic signature
```

`DS` (Delegation Signer) + `RRSIG` form a chain of trust: root vouches for `com.`'s key, so `com.`'s answers can be verified as genuine and unaltered.

The `google.com` delegation returned **no `DS` record** — only `NSEC3` records, which are authenticated proof that no DS exists, indicating the `google.com` zone itself is not DNSSEC-signed.

Set that against `DNSSEC=no/unsupported` from §5: signatures are published in the upper hierarchy, and this resolver checks none of them. **The security machinery exists and is off by default** — a theme that repeats with TLS verification (§9) and IAM (§12).

---

## 7 · Ports and listening services

*Covers networking slides 13 (HTTP/HTTPS) and 14 (ports)*

```bash
sudo ss -tlnp      # TCP listening + process
sudo ss -ulnp      # UDP listening + process
systemctl list-units 'wazuh*' --all
systemctl list-units --type=service --state=running | grep -Ei 'elastic|logstash|kibana|filebeat'
ps -p 645,654,1301 -o pid,args --no-headers
```

### Flags

| Flag | Meaning |
|---|---|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | **l**istening only |
| `-n` | **n**umeric — don't resolve names (also much faster) |
| `-p` | show owning **p**rocess and PID |

### Complete port map

| Port | Proto | Bind | Process | Service |
|---|---|---|---|---|
| 22 | TCP | `0.0.0.0` | `sshd` | SSH |
| 443 | TCP | `0.0.0.0` | `node` (636) | wazuh-dashboard |
| 1514 | TCP | `0.0.0.0` | `wazuh-remoted` (1459) | Agent → manager event stream |
| 1515 | TCP | `0.0.0.0` | `wazuh-authd` (1354) | Agent enrollment / key registration |
| 5601 | TCP | `0.0.0.0` | `MainThread` (586) | Kibana |
| 55000 | TCP | `0.0.0.0` | `python3` (1305) | Wazuh REST API |
| 9200 | TCP | `127.0.0.1` | `java` (645) | wazuh-indexer REST |
| 9300 | TCP | `127.0.0.1` | `java` (645) | wazuh-indexer cluster transport |
| 9201 | TCP | `127.0.0.1` | `java` (1301) | Elasticsearch REST |
| 9301 | TCP | `127.0.0.1` | `java` (1301) | Elasticsearch cluster transport |
| 9600 | TCP | `127.0.0.1` | `java` (654) | Logstash monitoring API |
| 6010 | TCP | `127.0.0.1` | `sshd` (3138) | X11 forwarding for my SSH session |
| 53 | TCP+UDP | `127.0.0.53`, `127.0.0.54` | `systemd-resolve` | DNS stub (§5) |
| 68 | UDP | `10.20.1.77` | `systemd-network` | DHCP client |
| 323 | UDP | `127.0.0.1` | `chronyd` | NTP time sync |

**Slide 14 taught three ports (22, 80, 443). This box has 18 listeners.**

### Reading the Local Address column

| Local Address | Means |
|---|---|
| `0.0.0.0:443` | All IPv4 interfaces — reachable off-box **if** the Security Group allows |
| `127.0.0.1:9200` | **Loopback only** — unreachable remotely no matter what the SG says |
| `[::]:22` | All IPv6 interfaces (often dual-stack, also accepting IPv4) |
| `[::ffff:127.0.0.1]:9200` | IPv4-mapped-IPv6 loopback — still loopback only |

> **The highest-value troubleshooting fact here:** if a service is bound to `127.0.0.1` and you can't reach it remotely, **opening the Security Group will not fix it.** You must change the service's bind address. Recognising this in five seconds instead of an hour in the AWS console is the skill.

### Service inventory — 7 services, two full stacks

```
wazuh-manager.service     active running
wazuh-indexer.service     active running
wazuh-dashboard.service   active running
elasticsearch.service     active running
filebeat.service          active running
kibana.service            active running
logstash.service          active running
```

Process identity resolved the ambiguity between the three JVMs:

```
 645  /usr/share/wazuh-indexer/jdk/bin/java ...   → 9200 / 9300
 654  /usr/share/logstash/jdk/bin/java ...        → 9600
1301  /usr/share/elasticsearch/jdk/bin/java ...   → 9201 / 9301
```

So this host runs **both** the Wazuh stack *and* a full ELK stack: two search engines side by side (the OpenSearch-derived wazuh-indexer on 9200, Elasticsearch on 9201) and two dashboards (wazuh-dashboard on 443, Kibana on 5601).

Three JVMs on 8 GiB — Logstash alone is `-Xmx1g`. This box is working hard, and `free -h` is worth watching.

---

## 8 · Firewall testing: refused vs timed out

*Covers security slide 6 (What is a Firewall?)*

Tested from an external machine against the public IP:

```bash
for p in 22 443 5601 55000 9200 80; do
  timeout 6 bash -c "cat < /dev/null > /dev/tcp/$IP/$p" 2>/dev/null \
    && echo "$p OPEN" \
    || { [ $? -eq 124 ] && echo "$p TIMED OUT" || echo "$p REFUSED"; }
done
```

| Port | Result | Interpretation |
|---|---|---|
| 22 | **OPEN** | SSH — intended |
| 443 | **OPEN** | wazuh-dashboard — intended |
| **5601** | **OPEN** | **Kibana — not intended. See §14** |
| 55000 | TIMED OUT | Security Group blocks it |
| 9200 | TIMED OUT | SG blocks it *and* it's loopback-bound |
| 80 | TIMED OUT | Nothing listening, SG blocks |

### The diagnostic that identifies the layer

Every blocked port **timed out**. None said "refused." That distinction is the fastest triage tool in networking:

| Symptom | Means | Where to look |
|---|---|---|
| `Connection refused` (instant) | Packet **reached** the host; nothing listening on that port | The **service** — down, or bound to loopback. Network is fine |
| `Connection timed out` (hangs) | Packet **silently dropped** in transit | Security Group, NACL, route table, host firewall |
| Works locally, not remotely | Bound to `127.0.0.1`, **or** SG closed | Check `ss` output first — faster than the console |

### Security Group vs Network ACL

| | Security Group | Network ACL |
|---|---|---|
| Attaches to | Network interface / instance | Subnet |
| **Stateful?** | **Yes** | **No** |
| Rules | Allow only | Allow **and** deny |
| Evaluation | All rules together | Numbered order, first match wins |
| Return traffic | Automatic | Needs an explicit rule |

**Statefulness is the concept that matters.** Because SGs are stateful, allowing inbound TCP/22 automatically permits the response packets out — no matching outbound rule needed. This is what §4's ICMP replies demonstrated. A stateless NACL needs rules in **both** directions, and forgetting the ephemeral port range (1024–65535) outbound is the classic NACL mistake.

---

## 9 · HTTP vs HTTPS

*Covers networking slide 13*

```bash
curl -I http://localhost:5601      # plain HTTP against a TLS port
curl -kI https://localhost         # HTTPS, verification skipped
curl -I https://localhost          # HTTPS, verification enforced
```

**Output:**

```
curl: (52) Empty reply from server
```

```
HTTP/1.1 302 Found
location: /app/login?
osd-name: ip-10-20-1-77
set-cookie: security_authentication=; Max-Age=0; ...; Secure; HttpOnly; Path=/
```

```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

| Command | Result | Why |
|---|---|---|
| `http://localhost:5601` | `(52) Empty reply` | TCP connected, then silence — **the port speaks TLS, not HTTP** |
| `https://localhost` + `-k` | `302 → /app/login` | Works; identity verification skipped |
| `https://localhost` no `-k` | `(60) unable to get local issuer certificate` | Self-signed cert — **the actual lesson** |

### What `-k` really silences

Error 60 is **not** "encryption failed." The TLS handshake succeeded; the channel was encrypted. What failed was **identity** — curl could not trace the certificate back to a trusted Certificate Authority, so it refused to trust the peer.

> **HTTPS = encryption + verified identity.** `-k` keeps the encryption and discards the identity check.

That's why `-k` is acceptable on your own box and dangerous anywhere else: an attacker impersonating the server would pass too.

Both dashboards require login — `443` returned `set-cookie: security_authentication=`, and `5601` returned `302 → /login?next=%2F`.

---

## 10 · Reading a TLS certificate

```bash
echo | openssl s_client -connect localhost:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
echo | openssl s_client -connect localhost:443 2>/dev/null | grep -E "Protocol|Cipher"
```

**Output:**

```
subject = C = US, L = California, O = Wazuh, OU = Wazuh, CN = wazuh-dashboard
issuer  = OU = Wazuh, O = Wazuh, L = California
notBefore = Aug 10 21:47:59 2026 GMT
notAfter  = Aug  7 21:47:59 2036 GMT

New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```

### Proof of self-signing

**Subject and issuer are the same organisation.** Wazuh issued a certificate vouching for Wazuh — no independent third party. That is the definition of self-signed, and precisely why §9 produced error 60.

**Second giveaway:** validity runs Aug 2026 → Aug 2036, a **10-year** window. Public CAs issue for 90 days to 13 months maximum. Ten years is only possible when signing your own.

### The cipher string contains all three encryption types

```
TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
                       │   │    │    │
                       │   │    │    └─ SHA384  → hashing (integrity)
                       │   │    └────── GCM     → authenticated mode
                       │   └─────────── 256-bit → key length
                       └─────────────── AES     → symmetric cipher
```

Security slide 9 lists symmetric, asymmetric and hashing as three separate things. A single live connection uses all three: **asymmetric** to verify identity and agree a session key, **AES symmetric** for the bulk data, **SHA384** for integrity.

Asymmetric for the introduction, symmetric for the conversation. TLS 1.3 is the current best version — and note the asymmetric part is absent from the cipher name, because TLS 1.3 dropped it: key exchange is always ECDHE now.

---

## 11 · The three encryption types

*Covers security slides 7, 8, 9*

### Hashing — one-way, proves integrity

```bash
echo "DevOps Day 2"  > /tmp/test.txt  && sha256sum /tmp/test.txt
echo "DevOps Day 2!" > /tmp/test2.txt && sha256sum /tmp/test2.txt
```

```
a9bcf5d565b2780c8979341119171cb9a58fbf6430fc85baf03906430d89803b  /tmp/test.txt
7f8cd5bc31fd5bcb4513b5a23af6a262e6f118ad59a573831c8fa2b5936b4440  /tmp/test2.txt
```

One character added, and **not a single character of the hash is recognisable.** That's the **avalanche effect** — why hashes detect tampering, and why the output reveals nothing about the size of the change.

Both are exactly 64 hex characters = 256 bits. **Fixed length regardless of input** — hash a 10 GB file, still 64 characters. Which is why it's one-way: the original cannot fit back out of 256 bits.

### Symmetric — one password locks and unlocks

```bash
openssl enc -aes-256-cbc -pbkdf2 -in /tmp/test.txt -out /tmp/test.enc -pass pass:MySecret123
cat /tmp/test.enc
openssl enc -d -aes-256-cbc -pbkdf2 -in /tmp/test.enc -pass pass:MySecret123
```

```
Salted__z▒5|▒▒=▒▒eI9n▒`ߔ▒▒h▒        →  decrypts to  "DevOps Day 2"
```

The same password locked and unlocked it. That's the definition — and the weakness: the password must reach the other party over some channel.

**`Salted__`** is a random value mixed in before encryption. Without it, identical plaintext + identical password would always produce identical ciphertext, letting an attacker spot repeats and use precomputed tables. Re-run the command and the bytes differ for the same input.

### Asymmetric — a keypair; the public half is safe to share

```bash
ssh-keygen -t ed25519 -f /tmp/day2key -N "" -C "day2-lab"
ls -l /tmp/day2key*
```

```
SHA256:r9rSCNZ52cMfGVJiadrqUIqUT4g7dkIb63l+AOnbwkw  day2-lab

-rw-------  399  /tmp/day2key       ← 600, private
-rw-r--r--   90  /tmp/day2key.pub   ← 644, public

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILCqbAd0/aIf98gdvH0vBYfXFCMYRIWYgdefwS0N0QU6 day2-lab
```

**`ssh-keygen` set `600` on the private key unprompted.** Nothing told it to — it knows SSH refuses a private key readable by other users. Day 1's `chmod 600` lesson appearing as a built-in safety rather than a convention. The public key got `644`, deliberately world-readable, because sharing it is the point.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

**And note the fingerprint** — `SHA256:r9rSCNZ...` is a **hash of the public key**. Hashing used to identify an asymmetric key. The three types interlock; they are not alternatives.

### Summary

| Type | Keys | Reversible? | Solves | Example here |
|---|---|---|---|---|
| **Symmetric** | One shared secret | Yes, with the key | Fast bulk encryption | `AES_256_GCM` in the TLS session |
| **Asymmetric** | Public + private pair | Yes, with the private key | Key exchange, identity, signatures | SSH login, TLS handshake |
| **Hashing** | None | **No — one-way** | Integrity, fingerprints, password storage | `SHA384` in the cipher, key fingerprint |

Encryption is reversible with a key. Hashing can never be undone. Conflating the two is a common interview stumble.

---

## 12 · IAM roles, authentication vs authorization

*Covers security slides 5 and 10*

```bash
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
aws sts get-caller-identity
id
sudo -l | tail -5
```

**Output:**

```
RansomwareResponseAutomationRole
```

```json
{
  "UserId": "AROARVAG67J44EKIGGN45:i-0d54xxxx",
  "Account": "1138xxxxxxxx",
  "Arn": "arn:aws:sts::1138xxxxxxxx:assumed-role/RansomwareResponseAutomationRole/i-0d54xxxx"
}
```

```
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)

User ubuntu may run the following commands on ip-10-20-1-77:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

### Authentication vs authorization, on both layers

| | Authentication — *who you are* | Authorization — *what you may do* |
|---|---|---|
| **Linux** | `uid=1000(ubuntu)` | `groups=...27(sudo)` → sudoers `(ALL:ALL) ALL` |
| **AWS** | `assumed-role/RansomwareResponseAutomationRole` | The IAM policies attached to that role |

Slide 5's distinction, live. Being `ubuntu` is authentication. Membership of group `27(sudo)` is authorization. Two independent systems — you can be fully authenticated and permitted nothing.

Also `groups=...4(adm)` grants read access to system logs — appropriate for a SIEM host.

### The important word is `assumed-role`

```
arn:aws:sts::1138xxxxxxxx:assumed-role/RansomwareResponseAutomationRole/i-0d54xxxx
                           ↑                                            ↑
                     not "user"                        instance ID as the session name
```

**There are no AWS access keys anywhere on this machine.** `aws configure` was never run. The instance fetches **temporary** credentials from `169.254.169.254`, and AWS rotates them automatically before expiry.

That is what an IAM role is (slide 10): identity granted to the **machine**, not stored as a secret. Nothing to leak in a git commit, nothing to rotate by hand.

`UserId` beginning `AROA` also identifies a role. IAM users begin `AIDA`.

### Why IMDSv2 exists — and it connects to slide 11

The `PUT`-for-a-token requirement from §2 now makes sense: **that endpoint hands out live AWS credentials.**

Under IMDSv1, any process that could be tricked into fetching a URL could read them — no token, no authentication, just a `GET`. That is not theoretical: it is the mechanism behind the **Capital One breach**, the very case cited on security slide 11 (*"In 2019, Capital One's cloud database was hacked, exposing millions of customers' details because of a misconfigured firewall"*). An SSRF flaw let an attacker reach the metadata service and steal the instance role's credentials.

Requiring a `PUT` first defeats it, because the flaws being exploited could only issue `GET` requests.

### One thing worth noticing about sudo

```
(ALL) NOPASSWD: ALL
```

Full root, no password prompt. Standard for cloud AMIs — the `ubuntu` account has no password at all, so there's nothing to type.

Which means **the SSH private key is the only thing between the internet and root on this box.** Not a key *plus* a password. Just the key. That is why `chmod 600` is enforced rather than suggested, and why §11's automatic `600` matters.

---

## 13 · Troubleshooting ladder

The reusable method distilled from this lab. When something doesn't work, walk **up** the stack — never guess in the middle.

| # | Question | Command |
|---|---|---|
| 1 | Is my interface up with an IP? | `ip -brief addr show` |
| 2 | Do I have a route out? | `ip route get 8.8.8.8` |
| 3 | Can I reach anything by IP? | `ping -c 3 8.8.8.8` |
| 4 | Does DNS resolve? | `dig +short google.com` |
| 5 | Is the remote port open? | `nc -zv <host> <port>` |
| 6 | Is the service listening, and on what address? | `sudo ss -tlnp` |
| 7 | Does the application answer? | `curl -v <url>` |
| 8 | Is TLS valid and unexpired? | `openssl s_client -connect <host>:443` |

**Steps 1–5 are the network. Steps 6–8 are the service.** Establishing which half you're in *before* changing anything is the entire skill.

---

## 14 · Findings

Two real issues surfaced by the lab, not lab artifacts.

### 1. Kibana reachable from the internet — port 5601

`ss` showed `0.0.0.0:5601`, and the external test in §8 returned **OPEN**. Kibana is TLS-enabled and does present a login page, so this is less severe than plaintext with no auth — but a SIEM dashboard should not be internet-facing at all. It broadens the attack surface for credential stuffing and any future Kibana CVE.

**Fix:** remove the inbound `5601` rule from the Security Group (EC2 → Security Groups → Inbound rules). Keep 22 and 443. Then reach Kibana over an SSH tunnel, run **from the local machine**:

```bash
ssh -i /path/to/key.pem -L 5601:localhost:5601 ubuntu@<public-ip>
```

Browse to `http://localhost:5601`. Encrypted inside SSH, nothing exposed.

Also worth reviewing: `55000` (Wazuh API), `1514` and `1515` are all bound to `0.0.0.0`. The SG currently blocks 55000; 1514/1515 must stay reachable for agents, but should be restricted to known agent CIDRs rather than `0.0.0.0/0`.

### 2. Self-signed certificates on both dashboards

Subject and issuer both `O = Wazuh`, 10-year validity. Functional but unverifiable — it's why every `curl` needs `-k`, and it trains users to click through browser warnings, which is exactly the habit that makes real interception attacks work.

**Fix for a production host:** issue a proper certificate via Let's Encrypt or AWS Certificate Manager against a real DNS name.

### Correction logged

My Day 1 write-up described the stack as Wazuh Manager + Elasticsearch + Logstash + Wazuh Dashboard. I initially believed this was wrong, since Wazuh 4.x ships **wazuh-indexer** (an OpenSearch fork) and **Filebeat** rather than Elasticsearch and Logstash.

`systemctl` and `ps` settled it: this host runs **both** stacks — `wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`, **and** `elasticsearch`, `logstash`, `kibana`, `filebeat`. The Day 1 description was accurate.

The lesson kept: **verify component names against the running system, not against memory or documentation.** `systemctl list-units` and `ps -p <pid> -o args` are the authority.

---

## 15 · Key takeaways

1. **`sudo ss -tlnp` is the highest-value networking command on a Linux server.** It answers "what is running and who can reach it" in one line. This box has 18 listeners; the lecture mentioned 3.
2. **Bind address beats firewall rule.** A service on `127.0.0.1` is unreachable remotely no matter how open the Security Group is. Check `ss` before touching the AWS console.
3. **Refused vs timed out identifies the layer.** Refused = reached the host, no listener → the service. Timed out = dropped in transit → the network.
4. **Security Groups are stateful; NACLs are not.** ICMP replies returned with no inbound rule, which proves it.
5. **`via` in the routing table is the router.** Present = hand it to the gateway. Absent = deliver it directly via ARP.
6. **TTL is a free distance measurement** on every ping. 64 means directly attached.
7. **DNS is a hierarchy plus caches, not a lookup table.** 13 roots → 13 TLD → 4 authoritative, and each cache holds the answer until its TTL expires. That's what "propagation" means.
8. **One hostname can have many correct answers.** GeoDNS returned different IPs to the local resolver and to `+trace`.
9. **HTTPS = encryption + verified identity.** `-k` keeps the first and discards the second. Error 60 is an identity failure, not an encryption failure.
10. **Encryption at one layer doesn't protect the layer below.** DNS queries here travel in plaintext with no DNSSEC, revealing every site visited before TLS ever starts.
11. **The three encryption types interlock.** `TLS_AES_256_GCM_SHA384` uses symmetric, asymmetric and hashing simultaneously.
12. **IAM roles remove secrets from disk.** `assumed-role` means temporary, auto-rotated credentials — and it's why IMDSv2 requires a token, given Capital One.
13. **Security machinery ships disabled.** DNSSEC off, TLS verification bypassable with one flag, IMDSv1 historically wide open. Defaults are convenience, not safety.
14. **Verify against the running system.** My own correction was wrong; `systemctl` and `ps` settled it.

---

## Deliverables

- [x] Watched `Day 2_Networking`
- [x] Watched `Day 7_Intro Security and Encryption`
- [x] Captured real output for all 12 lab steps
- [x] Built the port table from own `ss -tlnp` output
- [x] Confirmed actual service inventory via `systemctl` + `ps`
- [x] Corrected the Day 1 component question with evidence
- [x] Verified private key permissions are `600`
- [x] Identified 2 real security findings
- [x] Close the `5601` Security Group rule
- [x] Commit this file to `devops-20-day-journey`
- [ ] Post the Day 2 LinkedIn update

---

*Day 2 of 20 · [devops-20-day-journey](https://github.com/fahadaliseemab/devops-20-day-journey)*
