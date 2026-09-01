---
title: A First Look at MCP and Streaming Telemetry, on My Own EVPN/VXLAN Lab
date: 2026-08-25
draft: true
tags:
  - ansible
  - mcp
  - telemetry
  - gnmi
  - evpn-vxlan
  - arista
summary: Three new-to-me things in one afternoon — idempotent Ansible against a real EVPN/VXLAN fabric, a minimal read-only MCP server for network gear, and live gNMI telemetry through Prometheus and Grafana.
---

I've spent 21 years around routers and switches, most of it hands-on with Cisco Nexus data center gear. What I *haven't* done much of, until recently, is treat the network as something code talks to — idempotent config pushes, MCP servers, streaming telemetry instead of polling. This post is a "hi, bye" tour of three things I tried for the first time today, against a real lab rather than slides.

## The lab

A 1-spine/2-leaf EVPN/VXLAN fabric running on Arista cEOS, inside containerlab, on a Proxmox VM. eBGP underlay, EVPN overlay between the leaves, VLAN 10 over VNI 10010. Nothing exotic, but real enough that "idempotent" and "live telemetry" mean something rather than being buzzwords.

## 1. Idempotent automation, proven three times

The Ansible playbook pushes a management ACL to both leaves using the `arista.eos.eos_acls` resource module with `state: merged`:

```yaml
- name: Ensure LEAF-MGMT-ACL exists with defined rules
  arista.eos.eos_acls:
    config:
      - afi: ipv4
        acls:
          - name: LEAF-MGMT-ACL
            aces:
              - sequence: 10
                grant: permit
                source:
                  address: 172.20.20.0
                  wildcard_bits: 0.0.0.255
                destination:
                  any: true
                protocol: tcp
                protocol_options:
                  tcp:
                    established: true
    state: merged
```

The point isn't the ACL itself, it's the three-pass proof: `--check --diff` shows the intended change without touching anything, the real run applies it (`changed=2`), and running it again shows `changed=0`. That third run is the whole idea of idempotency in one number — the module gathers facts first, diffs against what I declared, and only touches what's actually different.

## 2. A minimal, read-only MCP server

MCP (Model Context Protocol) lets an LLM call tools directly. For network gear, that's genuinely useful *and* genuinely risky if you're not careful — so I built the smallest possible version, hardcoded to my three lab devices, with no write capability in the tool at all:

```python
@mcp.tool()
def show_bgp_summary(device: str) -> str:
    """Get 'show ip bgp summary' output for a lab device."""
    return _eapi(device, "show ip bgp summary")
```

That's it — two tools (`show_version`, `show_bgp_summary`), both read-only, both restricted to a hardcoded dict of lab IPs so there's no way to point it at anything else even by mistake. Wired up to Claude Desktop, I can now just ask "what's leaf1's BGP summary" and get a real answer pulled live from the fabric. Read-only-first, sandboxed-first — that's deliberate, not a shortcut I plan to skip past before I'd trust this pointed at anything real.

## 3. Streaming telemetry, not polling

The old model is SNMP polling a MIB every N seconds. gNMI flips that: you subscribe to a path and the device streams updates to you. I wired up:

```
Arista cEOS (gNMI) → gnmic (collector, Prometheus output) → Prometheus (scrape) → Grafana (dashboard)
```

gnmic subscribes to `/interfaces/interface/state/counters` on leaf1 and exposes what it collects on a `/metrics` endpoint; Prometheus scrapes that every 5 seconds; Grafana graphs it. The moment that actually clicked for me: watching a counter graph move in near-real-time from background BGP/EVPN keepalive traffic alone, with nothing else running on the fabric. That's a genuinely different feel from `show interface counters` in a terminal.

## What's next

Reconciling this with the fuller Terraform/Ansible/Nornir/GitLab CI pipeline I've been building separately, and eventually pointing the MCP server at something slightly less toy — still read-only, still sandboxed, but maybe with a couple more useful tools. More on that as it happens.
