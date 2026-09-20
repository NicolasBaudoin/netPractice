*This project has been created as part of the 42 curriculum by nbaudoin.*

# NetPractice

## Description

NetPractice is a hands-on introduction to computer networking. Instead of writing
code, you configure small simulated TCP/IP networks until they work.

Each of the 10 levels shows a broken network diagram: hosts, switches, routers and
sometimes the Internet, wired together. One or more objectives are listed at the top
of the page (for example "A must reach B"). Only some fields are editable — the
unshaded ones — and you must fill them with correct IP addresses, subnet masks,
default gateways and static routes until every objective turns green.

The difficulty ramps up across the levels:

| Levels | What they cover |
| --- | --- |
| 1–2 | Two hosts on a wire: same subnet, valid host addresses |
| 3 | A switch: several hosts in one broadcast domain |
| 4 | A router interface as a member of a LAN |
| 5 | Default gateways and the routing table of a host |
| 6 | Reaching the Internet, private vs. public addressing |
| 7 | Two routers, routes in both directions |
| 8 | Subnetting with non-trivial masks (/26 and friends) |
| 9–10 | Full topologies: several LANs, two routers, Internet, many goals at once |

The networks are simulated in the browser — nothing touches your real network stack.

## Instructions

### Running the training interface

1. Extract the archive provided on the project page:

   ```bash
   tar xzf net_practice.1.9.tgz
   cd net_practice
   ```

2. Launch the local web server and open the interface:

   ```bash
   ./run.sh
   ```

   `run.sh` picks a free port, starts `python3 -m http.server` and opens your default
   browser on it. Press `Ctrl-C` to shut the server down.

3. If `run.sh` does not work, do it manually:

   ```bash
   python3 -m http.server 49242
   ```

   then browse to `http://localhost:49242` (any free port works).

A local web server is required: browser security policies prevent the pages from
loading correctly through `file://`.

### Using the interface

- **Enter your 42 login** in the field on the home page. This is mandatory — an
  exported configuration without the right login is not accepted.
- The **"evaluation"** tab generates a random configuration, which is what is used
  during the defense.
- **[Check again]** validates the current configuration against the level's goals.
- **[Get my config]** downloads the configuration file for the current level.
- The **logs at the bottom of the page** explain *why* a configuration fails
  (invalid netmask, missing gateway, packet not for me, loop detected, ...). Read
  them: they are the fastest debugging tool in the project.

### Exporting configurations

Before moving on to the next level, click **[Get my config]** and keep the
downloaded file. Doing it level by level avoids having to replay everything at the
end.

### Submission

- Push your work to your Git repository; only what is in the repository is
  evaluated.
- **The 10 exported configuration files — one per level — must be placed at the
  root of the repository.**
- Double-check the file names before the defense.
- During the defense you must solve **three randomly chosen levels** within a
  limited time.
- **No external tools are allowed during the evaluation.** A simple calculator such
  as `bc` is tolerated, and that is the limit — so the binary arithmetic has to be
  in your head.

## Networking concepts studied

- **OSI / TCP/IP layers** — where addressing (layer 3) sits compared to switching
  (layer 2), and why a switch needs no IP address.
- **IPv4 addressing** — dotted-decimal notation, the 32-bit value behind it, the
  reserved ranges (`127.0.0.0/8` loopback, addresses above `223.x.x.x`).
- **Subnet masks** — contiguous-ones masks, dotted-decimal and CIDR notations,
  computing the network address and the broadcast address, deducing the usable host
  range and the number of hosts in a subnet.
- **Subnetting** — splitting an address block into smaller subnets (/25, /26, /30...)
  and checking that two subnets do not overlap.
- **Broadcast domains** — what a **switch** does (extends a single subnet, forwards
  to every port) versus what a **router** does (joins two different subnets, one
  interface and one IP per subnet).
- **Default gateway** — why a host needs one to leave its own subnet, and why the
  gateway address must belong to the host's own subnet.
- **Routing tables and static routes** — destination/mask plus next-hop gateway,
  longest-prefix intuition, the default route `0.0.0.0/0`, and the fact that the
  routes of this simulator are evaluated **in order, first match wins** — so the
  default route must come last.
- **Private vs. public addressing** — RFC 1918 ranges (`10.0.0.0/8`,
  `172.16.0.0/12`, `192.168.0.0/16`) and the fact that they are not routed over the
  Internet.
- **Common failure modes** — duplicate IP addresses, using the network or broadcast
  address on an interface, non-contiguous masks, routing loops, asymmetric routing
  (the request reaches the destination but the reply has no way back).

A companion guide with the theory and a step-by-step method is available in
[NETWORK-GUIDE.md](NETWORK-GUIDE.md).

## Resources

### Documentation and articles

- [RFC 791 — Internet Protocol](https://www.rfc-editor.org/rfc/rfc791) — the original IPv4 specification.
- [RFC 1918 — Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918) — the private address ranges.
- [RFC 4632 — CIDR](https://www.rfc-editor.org/rfc/rfc4632) — classless addressing and route aggregation.
- [Cloudflare — What is a subnet?](https://www.cloudflare.com/learning/network-layer/what-is-a-subnet/)
- [Cloudflare — What is the OSI model?](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/)
- [IBM — Subnet mask reference table](https://www.ibm.com/docs/en/i/7.5?topic=routing-subnet-mask)

### Tutorials and tools

- [Practical Networking — Subnetting Mastery](https://www.practicalnetworking.net/stand-alone/subnetting-mastery/) — the fastest way to learn to subnet mentally.
- [Subnetting Practice](https://subnettingpractice.com/) — drills, useful because no tool is allowed during the defense.
- [ipcalc](https://jodies.de/ipcalc) — to check your computations while training (not during the evaluation).

### Use of AI

AI (Claude) was used for:

- creating the readme
- search usefull ressurces to learn network
-
