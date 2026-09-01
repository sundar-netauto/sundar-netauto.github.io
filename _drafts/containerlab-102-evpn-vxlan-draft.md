---
title: "Containerlab 101: Network Labs as Code, Live-Debugged"
date: 2026-08-27
draft: false
tags: ["containerlab", "docker", "arista", "network-labs", "evpn-vxlan", "homelab"]
summary: "What containerlab actually is, the handful of commands that matter (deploy/inspect/destroy), how topology files and node 'kinds' work, and a cheat sheet of real gotchas from running a live Arista cEOS lab — captured while actually hitting every one of them."
---

I've had a containerlab-based EVPN/VXLAN lab sitting on a Proxmox VM for a while, and I could run the deploy command, get a fabric of Arista switches up, and poke around — but if you'd asked me to explain what containerlab was actually doing under the hood, or to troubleshoot it without a working example in front of me, I'd have struggled. This post is the "actually understand it" pass: what containerlab is, the commands that matter day to day, and a cheat sheet of gotchas pulled straight from a real, live debugging session rather than a clean happy-path tutorial. The network design (EVPN/VXLAN, symmetric IRB) is just the vehicle here — the point of this post is containerlab itself. If you want the networking deep-dive, that's a separate post.

## What containerlab actually is

[Containerlab](https://containerlab.dev/) is a tool that turns a YAML file into a running network topology. You describe nodes (routers, switches, hosts) and the links between them; containerlab creates a Docker container for each node, wires up virtual point-to-point links between their interfaces using Linux network namespaces, and hands you a fully cabled lab in about a minute. No physical hardware, no full VMs per node — every node is a container, which is why you can stand up a 5-node fabric without your laptop breaking a sweat.

Critically, containerlab isn't a network emulator itself — it doesn't know anything about BGP or VXLAN. It's an orchestrator. The actual "network OS" running inside each container is a real vendor image (in my case, Arista's cEOS-lab, a containerized build of real EOS) or something generic like a plain Linux container. Containerlab's job stops at: build the containers, wire the links, hand you a lab. Everything that happens after that — routing protocols converging, VLANs bridging, whatever — is entirely up to whatever's actually running inside the container.

## Prerequisites

Containerlab needs Linux network namespaces to build the virtual links, so it needs a Linux host — it won't run natively on macOS or Windows. My setup is a dedicated Proxmox-hosted Linux VM; if you don't have that, a free option on a Mac is a [Multipass](https://multipass.run/) or [UTM](https://mac.getutm.app/) VM. Inside that Linux host you need Docker (containerlab's actual runtime) and containerlab itself:

```bash
curl -fsSL https://get.docker.com | sh
bash -c "$(curl -sL https://get.containerlab.dev)"
```

For vendor images like Arista's cEOS-lab, you also need the image itself — it's not on Docker Hub, since it's licensed software. Register a free account at [arista.com](https://www.arista.com/), download the cEOS-lab `.tar.xz` under Support > Software Downloads, then import it as a local Docker image:

```bash
docker import cEOS-lab-<version>.tar.xz ceos:4.36.2F
```

The tag you choose here (`ceos:4.36.2F`) is what you'll reference as the `image:` in your topology file.

## Anatomy of a topology file

Everything containerlab does starts from one YAML file, conventionally named `*.clab.yml`. Here's the actual one behind my lab, trimmed to the structure that matters:

```yaml
name: evpn-vxlan-lab

topology:
  nodes:
    spine1:
      kind: arista_ceos
      image: ceos:4.36.2F
      startup-config: configs/spine1.cfg

    leaf1:
      kind: arista_ceos
      image: ceos:4.36.2F
      startup-config: configs/leaf1.cfg

    hostA:
      kind: linux
      image: alpine:latest
      exec:
        - ip addr add 10.10.10.10/24 dev eth1
        - ip route replace default via 10.10.10.1 dev eth1

  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["leaf1:eth2", "hostA:eth1"]
```

A few things worth understanding, not just copying:

**`kind`** tells containerlab which bootstrap logic to use for that node — how to wait for the node to be ready, where to inject a startup-config, what the management interface is called. This isn't cosmetic: `kind: arista_ceos` and the older `kind: ceos` are genuinely different in current containerlab, and using the wrong one for your containerlab version can silently affect behavior. If a topology file you find online uses `ceos` and yours needs `arista_ceos` (or vice versa), that's a version-drift issue, not a typo to "fix" blindly — check `containerlab version` against when the example was written.

**`startup-config`** points to a plain-text EOS config file that gets injected before the node boots, exactly as if you'd typed it at the console (or pasted it via `copy` in real life). This is the single most important thing to internalize: containerlab does not validate or understand what's inside that file. It just hands it to the node. If your config has a typo, or is simply missing something the design needs, containerlab will still report a clean, successful deploy — because from containerlab's point of view, nothing went wrong. The node booted, the links came up. Whatever's broken above that is entirely between you and EOS.

**`exec`** (only really relevant for simple `linux`-kind nodes) runs a list of shell commands inside the container right after it starts — I use it here to give the plain Alpine "host" containers an IP and a route without needing a full config file.

**`links`** is literally just a cabling list: `node:interface` to `node:interface`. Containerlab creates a veth pair for each entry and drops one end in each container's namespace.

## The core commands

Three commands cover almost everything you'll do day to day.

```bash
sudo containerlab deploy -t evpn-vxlan.clab.yml
```

This is the "bring it all up" command. Run from the directory containing the topology file (or pass a path). Real output looks like this:

```
04:49:57 INFO Containerlab started version=0.79.0
04:49:57 INFO Parsing & checking topology file=evpn-vxlan.clab.yml
04:49:57 INFO Creating docker network name=clab IPv4 subnet=172.20.20.0/24 ...
04:49:57 INFO Creating container name=leaf1
04:49:58 INFO Running postdeploy actions for Arista cEOS 'spine1' node
04:49:58 INFO Created link: spine1:eth1 ▪┄┄▪ leaf1:eth1
...
╭────────────────────────────┬───────────────┬─────────┬───────────────────╮
│            Name            │   Kind/Image  │  State  │   IPv4/6 Address  │
├────────────────────────────┼───────────────┼─────────┼───────────────────┤
│ clab-evpn-vxlan-lab-leaf1  │ arista_ceos   │ running │ 172.20.20.4       │
╰────────────────────────────┴───────────────┴─────────┴───────────────────╯
```

That final table is your map: every node gets an auto-assigned management IP on containerlab's own Docker network (`clab`, `172.20.20.0/24` by default), separate from whatever addressing your lab's own topology uses internally. The container naming convention — `clab-<lab-name>-<node-name>` — matters, because it's what you use for every subsequent `docker exec`.

"Running postdeploy actions for Arista cEOS" is containerlab waiting for the EOS management CLI inside the container to actually become responsive before moving on — cEOS booting a real EOS image takes noticeably longer than a plain Linux container coming up, so don't be surprised if a deploy takes 1-2 minutes rather than a few seconds once you have several EOS-kind nodes.

```bash
containerlab inspect -t evpn-vxlan.clab.yml
```

Reprints that same status table without redeploying anything — useful after you've walked away and come back, or from a different terminal.

```bash
sudo containerlab destroy -t evpn-vxlan.clab.yml --cleanup
```

Tears the whole lab down — containers, virtual links, the generated per-lab directory. Always use `--cleanup` before a redeploy you actually care about; without it, stale directories and network state can linger and cause a "fresh" deploy to behave oddly.

## Talking to the nodes

`docker exec` is how you actually reach into a running node, and for Arista images specifically there's a wrinkle worth knowing up front.

For a one-off, single command at privilege level 15 (equivalent to `enable`), use `FastCli`:

```bash
docker exec clab-evpn-vxlan-lab-leaf1 FastCli -p 15 -c "show ip bgp summary"
```

For multiple commands that need to build on each other (like entering config mode), don't chain multiple `-c` flags on `Cli` or `FastCli` expecting them to share session state — in my experience they don't reliably (each `-c` can behave like it's starting fresh, so an `enable` in one `-c` doesn't carry over to the next). Instead, pipe a real multi-line session in via stdin, which behaves exactly like typing at an interactive prompt:

```bash
docker exec -i clab-evpn-vxlan-lab-leaf1 Cli << 'EOF'
enable
configure terminal
ip routing
end
write memory
EOF
```

For plain Linux-kind nodes (like the Alpine `hostA`/`hostB` in this lab), it's just a normal container, so ordinary `docker exec` works as you'd expect:

```bash
docker exec clab-evpn-vxlan-lab-hostA ping -c 4 10.10.20.10
```

## Gotchas and lessons learned (the hard way)

Everything below is a real thing I hit while bringing this specific lab up, not a hypothetical list.

**A "successful" deploy proves nothing about the network design working.** Containerlab's job ends at "containers are up, links are wired." I had a fully green deploy with all nodes reporting `running`, and the fabric still had zero working control plane, because the startup-configs were missing pieces. Deploy output tells you the orchestration succeeded, not that your network works.

**EOS defaults to `no ip routing`.** Arista's heritage is as a switch, so even with a full `router bgp` configuration in your startup-config, nothing routes until you explicitly add `ip routing` (for the default VRF) and, separately, `ip routing vrf <name>` for any non-default VRF. Missing this looked, from the outside, exactly like "BGP is disabled" — check this early whenever a configured routing protocol just isn't coming up.

**A live CLI fix doesn't persist unless you edit the actual startup-config file.** I fixed a routing issue via `write memory` inside a running node, confirmed it worked, then did a full `destroy`/`deploy` later and watched the exact same bug reappear — because `write memory` only updates that container's own running state, not the source `.cfg` file on disk that containerlab reads on the next deploy. If a fix needs to survive a redeploy, it has to go into the file.

**Verify a file write actually landed before trusting the next step.** A heredoc write into a config file silently produced an empty file once (a copy-paste/terminal issue, not containerlab's fault) — and everything downstream behaved exactly as if the config had never been fixed, with no error pointing back at the real cause. A quick `cat` and `wc -l` after any config edit is cheap insurance.

**`Exited (137)` containers are stopped, not gone.** Docker's own exit code for SIGKILL — usually a VM reboot or resource pressure, not something containerlab did wrong. The container and its network wiring still exist in containerlab's bookkeeping; always `destroy --cleanup` before redeploying rather than assuming a fresh `deploy` will clean up after a crash on its own.

**A plain Linux container already has a default route.** Docker gives every container a default route out its management interface (`eth0`) automatically. If your topology's `exec:` block tries `ip route add default via ...` on a second interface, it'll fail with `RTNETLINK answers: File exists` — because you can't have two default routes in one table. Use `ip route replace default via ...` instead; it overwrites whichever default route already exists rather than erroring.

**"Up" doesn't mean "in the routing table yet."** Right after a fresh deploy, an interface can report `up, line protocol is up` while the actual routing table for that VRF is still empty for another minute or two — cEOS runs interface state, routing protocols, and RIB programming as separate internal processes that take a moment to sync. Don't read an empty `show ip route` immediately post-deploy as broken; recheck a minute later before concluding anything.

**A command that runs cleanly (no `% Invalid input`) can still tell you nothing useful.** `show vxlan vtep` is a real, valid EOS command — but it tracks VTEPs discovered for L2 flood-domain replication, not routed (EVPN Type-5/symmetric-IRB) traffic. In a lab where two leaves don't share a VLAN, it can correctly show zero peers *while the actual routing works perfectly* — I chased it as a symptom for a while before realizing it simply wasn't the right check for this design. The commands that actually mattered were `show bgp evpn route-type ip-prefix` (is the route valid in BGP?) and `show ip route vrf <name>` (did it actually get installed?).

**Extended communities aren't automatically sent to eBGP peers.** This was the real, final bug: a route-target-based EVPN route was showing up correctly on both leaves (import filtering worked), but the "Router MAC" extended community attached to it — needed for the receiver to actually know which VTEP/MAC to forward toward — was silently not making it across. Adding `neighbor <ip> send-community extended` under `router bgp` fixed it immediately. Route-target itself seems to get special-cased for EVPN's own import logic even without this, which is exactly what made the symptom confusing — everything about route acceptance looked fine right up until the point of actually installing a forwarding path.

## Quick reference

| Task | Command |
|---|---|
| Deploy a lab | `sudo containerlab deploy -t <file>.clab.yml` |
| Check status without redeploying | `containerlab inspect -t <file>.clab.yml` |
| Tear down cleanly | `sudo containerlab destroy -t <file>.clab.yml --cleanup` |
| One-shot privileged EOS command | `docker exec <node> FastCli -p 15 -c "<cmd>"` |
| Multi-command EOS session (config mode etc.) | `docker exec -i <node> Cli << 'EOF' ... EOF` |
| List all containers for a lab | `docker ps -a --filter "name=clab-<lab-name>"` |
| See what a Linux-kind node sees | `docker exec <node> ip route show` |

And the two shell functions I ended up wrapping around these, once I got tired of retyping full container names:

```bash
# ~/.bashrc
ceos() { docker exec "clab-evpn-vxlan-lab-$1" FastCli -p 15 -c "$2"; }
chost() { docker exec "clab-evpn-vxlan-lab-$1" sh -c "$2"; }
```

Usage: `ceos leaf1 "show ip bgp summary"`, `chost hostA "ping -c 4 10.10.20.10"`.

## What's next

This post is deliberately just containerlab mechanics. The next one picks up the actual network design — going from a single flat VLAN to real symmetric-IRB inter-subnet routing (VRFs, L3VNIs, anycast gateways) — and the one after that is the full troubleshooting story from today, in order, with the real command output at each step.
