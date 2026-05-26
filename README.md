# Lab: Check Network Connectivity with `ping` and `traceroute`

- **Series:** linux-ops-mastery — RHCSA Networking
- **Subjects covered:** ICMP echo requests (`ping`), round-trip latency and loss, `ping -c` count limits, IPv4 vs IPv6 selection, `traceroute` TTL-expiry path discovery, asymmetric paths, installing `traceroute` on minimal RHEL installs
- **Career arcs covered:** RHCSA (basic connectivity proof on EX200), RHCE (pre-flight checks in playbooks), SRE (first 60 seconds of "site down"), DevOps (CI runner cannot reach artifact server), AI/MLOps (distributed training job stuck on rank-0 NFS mount)
- **Prerequisite:** Working default route and DNS (Labs 31 or 35); outbound ICMP not blocked by security policy
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 foundation · 2–3 `ping` patterns · 4–5 `traceroute` interpretation · 6 exam-realistic capstone

---

## Objective

Turn vague "the network feels bad" reports into **repeatable measurements**. By the end of this lab you can prove local stack health with targeted pings, quantify latency and packet loss, bound a test with `-c`, and map the **forward path** toward a remote host with `traceroute` — understanding what the hop list does and does not guarantee.

The capstone is the RHCSA-realistic prompt: *"Verify the host can reach its default gateway, a well-known public resolver, and a public DNS name; capture five pings to the resolver; run `traceroute` to the same resolver and save the hop list to `/root/path.txt`."*

> **Lab safety note:** Some labs and clouds **block ICMP** toward the internet or between subnets. If `ping 8.8.8.8` fails but `curl` works, your environment is filtering — substitute **your gateway** and **an internal bastion** that permits ICMP.

---

## Concept: `ping` Proves Reachability; `traceroute` Sketches the Path

**`ping`** sends ICMP Echo Request messages and listens for Echo Reply. A successful exchange proves **L3 connectivity and return path viability** between two endpoints — at least for ICMP, which firewalls may treat differently than TCP.

**`traceroute`** sends probes with increasing TTL values so each router along the path returns **ICMP Time Exceeded** (or ICMP errors for UDP traceroute modes). The result is an **ordered list of responding hops** — useful, but not a contract: routers may rate-limit, lie, or appear duplicated.

```
   ┌────────────┐      ICMP echo request       ┌────────────┐
   │  Your RHEL │  ─────────────────────────► │  Target    │
   │  host      │  ◄─────────────────────────  │  (GW/DNS)  │
   └────────────┘      ICMP echo reply          └────────────┘

   traceroute idea (simplified):
   TTL=1  ─► first router returns "time exceeded"
   TTL=2  ─► second router returns "time exceeded"
   ...
   until probe reaches open port or max hops
```

> **Why this matters:** RHCSA and real operations both want **evidence**, not vibes. `ping -c 3 GATEWAY` is the fastest proof your default route points somewhere alive. `traceroute` is the fastest *human-readable* sketch when latency spikes mid-path.

---

## 📜 Why `ping` and Traceroute Exist — The Story

In August 1983, Mike Muuss at the US Army Ballistics Research Laboratory wrote `ping` as a diagnostic tool built on ICMP — the same control-plane protocol IP uses for unreachable messages and echo testing. The name was a sonic pun on submarine sonar **pings**, because the program sends a pulse and listens for a return.

ICMP echo became the **universal hello** of IP networks: if echo works, something is right; if echo fails, *something* is wrong — interface, cable, route, ACL, or policy. It does not tell you *which* layer failed, which is why operators pair `ping` with `ip route`, ARP tables, and TCP-level tests.

Path tracing tools evolved in parallel. Van Jacobson's classic **traceroute** on BSD used UDP datagrams or ICMP probes with increasing TTLs to force each hop to reveal itself. Linux distributions ship multiple implementations; RHEL provides `/usr/bin/traceroute` via the `traceroute` package on minimal systems where only `tracepath` might be present initially.

Modern networks complicate the story: **load balancing**, **MPLS tunnels**, and **firewall ICMP rate limits** produce star-fields of `*` hops that are not outages. Traceroute shows the **probe path**, not guaranteed symmetry with your application's TCP flows.

> **The point of the story:** `ping` answers "is there a path that carries my echo?" Traceroute answers "which routers touched my TTL-limited probes?" Neither replaces application-layer tests — but both remain the first page of every runbook.

---

## 👪 The Connectivity-Testing Family — Who Lives There

### ICMP and reachability tools

| Tool | Layer | What a success implies |
|---|---|---|
| `ping` | L3 ICMP (typical) | Two-way path for ICMP echo/reply |
| `ping6` | IPv6 ICMPv6 | Same role for IPv6 |
| `tracepath` | L3 path MTU + hops | Often preinstalled; UDP-less trace |
| `traceroute` | L3 hop discovery | Classic hop listing; flags vary by build |

### Common `ping` options on RHEL 9

| Flag | Meaning |
|---|---|
| `-c N` | Stop after `N` packets (script-friendly) |
| `-W SEC` | Per-reply wait cap (avoids hanging scripts) |
| `-I DEV` | Bind to source interface |
| `-4` / `-6` | Force address family |

### `traceroute` modes you may see

| Mode hint | Behavior sketch |
|---|---|
| Default | Often UDP high ports toward destination |
| `-I` | ICMP probes (needs capability) |
| `-T` | TCP SYN tracing (through some firewalls) |

> **The point of the family tree:** Pick **bounded** `ping` for automation (`-c`). Pick **traceroute** when the question is "where does latency begin?" — not merely "up or down?"

---

## 🔬 The Anatomy of `ping -c 5 8.8.8.8` Output — In One Diagram

```
$ ping -c 5 -W 2 8.8.8.8
  │    │    │    │     │
  │    │    │    │     └─ Target IPv4 (Google Public DNS — lab-friendly if allowed)
  │    │    │    └─ Wait up to 2 seconds per reply (non-interactive discipline)
  │    │    └─ Send exactly 5 echo requests then exit
  │    └─ Count flag
  └─ ICMP echo client

Typical summary lines (illustrative):
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=12.4 ms
...
--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 801ms
rtt min/avg/max/mdev = 11.2/12.0/13.1/0.7 ms
  │                                    │         │
  │                                    │         └─ jitter-ish spread (mdev)
  │                                    └─ average RTT — what SLO dashboards echo
  └─ loss percentage — anything non-zero deserves follow-up with traceroute + TCP test
```

> **Reading rule:** Always quote **transmitted vs received** and **loss %** in tickets — not a single RTT sample.

---

## 📚 Connectivity Checks — Reference Table

| Task | Command | Notes |
|---|---|---|
| Ping gateway thrice | `ping -c 3 GATEWAY` | Proves on-link path |
| Ping by DNS name | `ping -c 3 redhat.com` | Exercises DNS + IPv4 path |
| Bounded wait | `ping -c 3 -W 2 HOST` | Prevents hung CI scripts |
| Trace by ICMP | `sudo traceroute -I -n 8.8.8.8` | `-n` avoids slow reverse DNS |
| Save hop list | `traceroute -n HOST > /root/path.txt` | Inspect with `cat -n` |
| Install traceroute | `sudo dnf install -y traceroute` | Minimal images may lack it |

> **Rule one of connectivity labs:** If ICMP is blocked, document that fact and fall back to **TCP** tests (`curl`, `nc -vz`) — but master ICMP first; it is the exam's lingua franca.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Demonstrating connectivity to the grader-supplied gateway is a frequent partial credit step. |
| **RHCE candidate** | Ansible `wait_for` / `uri` modules replace ping in playbooks — same mental model: prove path before expensive tasks. |
| **SRE / Platform** | Outage bridges open with `ping` to gateway, `ping` to peer AZ, then `traceroute` to isolate provider loss. |
| **DevOps** | CI agents that cannot `ping` package mirrors waste hours; a three-packet check fails fast. |
| **AI / MLOps** | All-reduce hangs often correlate with **one rank** losing east-west ping — traceroute finds asymmetric return paths. |

---

## 🔧 The 6 Tasks

> Six phases that build **bounded measurements** and a **saved traceroute artifact**.

---

### Task 1 — Install `traceroute` and confirm `ping` basics

**Purpose:** Ensure both tools exist on a minimal RHEL 9 system and print local routing context.

```bash
sudo dnf install -y traceroute
ip -4 route show default
GATE=$(ip -4 route show default | awk '/default/ {print $3; exit}')
echo "DEFAULT_GW=${GATE}"
ping -c 1 -W 2 127.0.0.1
```

**Human-Readable Breakdown:** Install the `traceroute` package, display the IPv4 default route, capture the gateway field into `GATE`, and prove the local IP stack handles loopback echo.

**Reading it left to right:** `ip route` parses `via` for the gateway IPv4. `ping 127.0.0.1` never leaves the box — it validates ICMP client and kernel path.

**The story:** Loopback first is a **sanity ritual**. If loopback fails, the problem is local and weird — fix that before blaming the WAN.

**Expected output:**

```text
Complete!
default via 192.168.122.1 dev eth0 proto static metric 100
DEFAULT_GW=192.168.122.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.045 ms
--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

**Switches**

| Token | Meaning |
|---|---|
| `dnf install -y` | Non-interactive package install |
| `ip -4 route show default` | Show only default IPv4 route |
| `ping -c 1` | Single probe |
| `ping -W 2` | Wait at most two seconds for reply |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `No route to host` on loopback | Severe local misconfig — inspect `ip rule` / namespaces |
| `DEFAULT_GW` empty | No default route — revisit static/DHCP lab |
| `traceroute` already installed | `dnf` reports nothing to do — continue |

---

### Task 2 — Ping the default gateway with bounded count

**Purpose:** Prove the **on-link** path to the configured gateway without infinite pings.

```bash
ping -c 3 -W 2 "$GATE"
```

**Human-Readable Breakdown:** Send three ICMP echoes to the gateway extracted in Task 1, waiting at most two seconds per round — typical automation-friendly bounds.

**Reading it left to right:** `ping` reads `GATE` from the environment. `-c 3` terminates after three probes. `-W` caps per-packet wait.

**The story:** Gateway ping distinguishes **"I have no route"** from **"I have a route but the next hop is dead"** — the oldest partition in networking triage.

**Expected output:**

```text
PING 192.168.122.1 (192.168.122.1) 56(84) bytes of data.
64 bytes from 192.168.122.1: icmp_seq=1 ttl=64 time=0.214 ms
64 bytes from 192.168.122.1: icmp_seq=2 ttl=64 time=0.198 ms
64 bytes from 192.168.122.1: icmp_seq=3 ttl=64 time=0.201 ms
--- 192.168.122.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.198/0.204/0.214/0.007 ms
```

**Switches**

| Token | Meaning |
|---|---|
| `-c 3` | Exactly three probes |
| `-W 2` | Seconds to wait for each reply |
| `ttl=64` | Typical for same-LAN hop |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Destination Host Unreachable` | ARP/on-link failure — check `ip neigh`, link state |
| 100% packet loss | Gateway may block ICMP — try TCP `curl` to gateway HTTPS |
| Wrong gateway | Re-run `ip route`; fix NM profile |

---

### Task 3 — Ping a public resolver by IPv4 address

**Purpose:** Prove **beyond-local** L3 connectivity without involving DNS yet.

```bash
ping -c 5 -W 2 8.8.8.8
```

**Human-Readable Breakdown:** Send five probes to Google Public DNS IPv4 — a widely routed address useful when DNS itself is suspect.

**Reading it left to right:** If Task 2 succeeds but Task 3 fails, suspect **upstream filtering**, **missing default route**, or **policy routing** — not ARP.

**The story:** This is the classic **"is the internet up?"** check — crude, fast, and surprisingly effective on homelab networks.

**Expected output:**

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=12.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=11.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=118 time=12.1 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=118 time=12.0 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=118 time=12.2 ms
--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 11.9/12.1/12.4/0.18 ms
```

**Switches**

| Token | Meaning |
|---|---|
| `8.8.8.8` | Well-known recursive resolver IPv4 |
| `-c 5` | Five-sample RTT average |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| 100% loss but gateway works | Upstream ACL — try `1.1.1.1` or internal target |
| High jitter | Capture `mdev` — correlate with `traceroute` Task 4 |
| Only IPv6 works | You are on IPv6-only network — use `ping -6` |

---

### Task 4 — Run `traceroute` with numeric output and interpret stars

**Purpose:** Map hops toward `8.8.8.8` and learn to read `*` lines calmly.

```bash
traceroute -n -w 2 -q 1 -m 15 8.8.8.8 | head -n 20
```

**Human-Readable Breakdown:** Traceroute numerically (`-n`), two-second probe wait, single probe per hop (`-q 1`), cap max TTL at 15, pipe through `head` to avoid huge output in class.

**Reading it left to right:** Each line is a TTL bucket. Some hops respond with ICMP errors; some do not — `*` means **no response for that probe**, not always packet loss for your application.

**The story:** Traceroute is a **storytelling** tool: it turns latency mysteries into geography-of-routers sketches. Stars are normal in carrier cores.

**Expected output:**

```text
traceroute to 8.8.8.8 (8.8.8.8), 15 hops max, 60 byte packets
 1  192.168.122.1  0.204 ms
 2  10.0.2.2  0.518 ms
 3  * * *
 4  203.0.113.7  9.812 ms
 5  198.51.100.33  12.441 ms
 6  8.8.8.8  11.993 ms
```

**Switches**

| Token | Meaning |
|---|---|
| `-n` | No reverse DNS — faster |
| `-w 2` | Per-probe wait seconds |
| `-q 1` | One probe per hop (quieter) |
| `-m 15` | Maximum TTL / hop count |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| All stars | ICMP administratively prohibited — try `sudo traceroute -I` or TCP mode |
| `command not found` | Install `traceroute` (Task 1) |
| Hangs at first hop | Local firewall — verify `GATE` ping |

---

### Task 5 — Combine DNS proof with ping and traceroute to hostname

**Purpose:** Show that **name resolution** and **path tracing** are orthogonal checks.

```bash
getent ahostsv4 redhat.com | head -n 3
ping -c 3 -W 2 redhat.com
traceroute -n -w 2 -q 1 -m 20 redhat.com | head -n 15
```

**Human-Readable Breakdown:** Resolve `redhat.com` to IPv4 addresses, ping the name (libc picks an address), traceroute to the name (resolver again).

**Reading it left to right:** `getent` uses NSS; `ping` may pick a different A record than you expect — that is why listing addresses first matters.

**The story:** Junior admins blame DNS for routing issues and routing for DNS issues. This task keeps the experiments **sequential** so the failure domain is obvious.

**Expected output:**

```text
209.132.183.44  STREAM redhat.com
209.132.183.45  STREAM redhat.com
2600:1408:10::b81e:1348  STREAM
PING redhat.com (209.132.183.44) 56(84) bytes of data.
64 bytes from 209.132.183.44: icmp_seq=1 ttl=52 time=18.2 ms
64 bytes from 209.132.183.44: icmp_seq=2 ttl=52 time=17.9 ms
64 bytes from 209.132.183.44: icmp_seq=3 ttl=52 time=18.0 ms
--- redhat.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
traceroute to redhat.com (209.132.183.44), 20 hops max, 60 byte packets
 1  192.168.122.1  0.195 ms
 2  10.0.2.2  0.401 ms
 3  * * *
 4  198.51.100.2  10.112 ms
...
```

**Switches**

| Token | Meaning |
|---|---|
| `getent ahostsv4` | IPv4-focused address listing |
| `ping redhat.com` | Implicit resolver inside `ping` |
| `traceroute redhat.com` | Resolver runs again — watch for different A record |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ping` resolves but fails | Firewall path — compare with `curl -I` |
| `getent` empty | DNS broken — inspect `/etc/resolv.conf` and NM DNS |
| Different IPs between tools | Multiple A records — normal |

---

### Task 6 — Capstone: scripted proof file and saved traceroute

**Task statement:** *"Prove connectivity to the default gateway and to `8.8.8.8` with bounded pings; run a numeric `traceroute` to `8.8.8.8` and save the full output to `/root/path.txt`; leave no stray temporary files beyond that proof."*

**Purpose:** Produce **grader-friendly artifacts** — a hop file on disk and a transcript showing zero packet loss locally.

```bash
sudo -i
GATE=$(ip -4 route show default | awk '/default/ {print $3; exit}')
{
  echo "=== ping gateway ==="
  ping -c 5 -W 2 "$GATE"
  echo "=== ping 8.8.8.8 ==="
  ping -c 5 -W 2 8.8.8.8
} | tee /root/ping-proof.txt
traceroute -n -w 2 -q 1 -m 20 8.8.8.8 | tee /root/path.txt
wc -l /root/ping-proof.txt /root/path.txt
```

**Human-Readable Breakdown:** Become root, derive gateway, run two bounded ping sections captured to `/root/ping-proof.txt`, traceroute to `8.8.8.8` into `/root/path.txt`, and count lines for quick sanity.

**Layer stack you exercised:**

```text
User scripts (`tee`) → ICMP echo (`ping`) → kernel raw socket path → NIC driver
User scripts (`tee`) → TTL-limited probes (`traceroute`) → per-hop ICMP errors
```

**The story:** This is the **evidence bundle** you attach to tickets — not screenshots of one ping.

**Expected verification output:**

```text
=== ping gateway ===
PING 192.168.122.1 (192.168.122.1) 56(84) bytes of data.
64 bytes from 192.168.122.1: icmp_seq=1 ttl=64 time=0.210 ms
...
5 packets transmitted, 5 received, 0% packet loss, time 1024ms
=== ping 8.8.8.8 ===
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
...
5 packets transmitted, 5 received, 0% packet loss, time 4012ms
  42 /root/ping-proof.txt
  16 /root/path.txt
```

**Cleanup**

```bash
rm -f /root/ping-proof.txt /root/path.txt
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty `path.txt` | Traceroute lacked permission — rerun with `sudo` |
| `ping-proof.txt` shows loss | Focus on first failing section — gateway vs WAN |
| `tee` permission denied | Ensure `sudo -i` root shell |

---

## 🔍 Connectivity Testing Decision Guide

```
Something "cannot be reached"
  │
  ├── "Is my own IP stack alive?"
  │       └── ✅ ping -c 1 127.0.0.1
  │
  ├── "Is the default route sane?"
  │       └── ✅ ip -4 route show default
  │
  ├── "Does the next hop answer?"
  │       └── ✅ ping -c 3 GATEWAY
  │
  ├── "Does the wider internet answer ICMP?"
  │       └── ✅ ping -c 5 8.8.8.8
  │
  ├── "DNS or IP problem?"
  │       └── ✅ getent hosts NAME then ping NAME
  │
  └── "Where does latency or loss begin?"
          └── ✅ traceroute -n HOST
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Install `traceroute`; show default route; ping loopback
- [ ] 02 Ping default gateway with `-c` / `-W`
- [ ] 03 Ping `8.8.8.8` by address
- [ ] 04 `traceroute -n` toward `8.8.8.8` and read stars calmly
- [ ] 05 Resolve and ping `redhat.com`; traceroute hostname
- [ ] 06 Capstone evidence files + cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Unbounded `ping` in scripts | Hung automation | Always `-c` |
| Interpreting `*` as total outage | False panic | Retry with `-I` or TCP tools |
| Confusing RTT with bandwidth | Misleading conclusions | Throughput needs iperf/flent |
| Ignoring ICMP blocks | False negatives | Try `curl` / `nc` |
| Running traceroute without `-n` | Slow runs | Add `-n` |
| Assuming symmetric return path | Mystery asymmetry | Mention in ticket; use TCPTrace |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Practice quoting **loss percentage** and **avg RTT** from real output — exam tasks often ask for observed numbers.

**RHCE candidate**
- Show how Ansible `command: ping -c 1` is guarded with `changed_when: false`.

**SRE / Platform interview**
- Explain difference between **ICMP drop** and **TCP blackhole** — firewalls treat them differently.

**DevOps**
- Bake `ping -c 1` preflight into deploy scripts with clear stderr messages.

**AI / MLOps**
- When NCCL times out, include `ping` matrix between ranks in the bug report — ops teams love hop evidence.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 31 — Configure a Static IP Address | Ensures gateway/DNS exist to test |
| Lab 33 — Display IP and Routing Info | Interprets routes you are probing |
| Lab 34 — Inspecting Listening Sockets | Moves from outbound ICMP to inbound listeners |
| Lab 35 — Text-Based Network Config `nmtui` | Alternative path to working resolver settings |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
