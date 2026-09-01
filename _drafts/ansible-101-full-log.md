---
title: Ansible 101 — full troubleshooting log (personal reference, not for publishing)
lab: clab02.clab.yaml — router1, router2 (arista_ceos), host1 (linux) on netdevops-lab-vm02
---

# Ansible 101 — everything that actually happened

Reference doc only. The published post should be the simplified version,
this is the raw trail for when I want to write a deeper post later or
just remember what actually broke and why.

## Environment

- netdevops-lab-vm02, Ubuntu 22.04.5 LTS, kernel 5.15.0-190-generic
- containerlab lab `clab02.clab.yaml`: router1 + router2 (arista_ceos,
  ceos:4.36.2F), host1 (linux/alpine) — same lab from the containerlab-101 post
- Ansible run from inside netdevops-lab-vm02 directly (SSH into the VM,
  install Ansible there, target containerlab mgmt IPs/hostnames
  directly) rather than reaching the lab externally — simplest path,
  no extra routing setup needed
- Working dir: `~/ansible101/`

## Setup

**Python venv + Ansible:**
```bash
sudo apt update
sudo apt install -y python3-venv python3-pip
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible
ansible --version
```
Venv over a bare system-wide pip install — avoids fighting system
Python, and newer Debian/Ubuntu (23.04+/Debian 12+) refuse a bare
`pip install` outside a venv anyway (PEP 668 "externally managed
environment").

**Arista collection:**
```bash
ansible-galaxy collection install arista.eos
ansible-galaxy collection list | grep -E "arista|netcommon"
ansible-doc arista.eos.eos_command   # sanity check module is visible
```

**inventory.yaml:**
```yaml
all:
  children:
    routers:
      hosts:
        router1:
          ansible_host: clab-clab-02-router1
        router2:
          ansible_host: clab-clab-02-router2
      vars:
        ansible_connection: ansible.netcommon.network_cli
        ansible_network_os: arista.eos.eos
        ansible_user: admin
        ansible_password: admin123
        ansible_become: true
        ansible_become_method: enable
```
`ansible_host` uses the containerlab container names, not raw IPs —
containerlab adds these to `/etc/hosts` on every deploy, so the
inventory survives a lab redeploy even though IPs reshuffle.

**ansible.cfg:**
```ini
[defaults]
host_key_checking = False
inventory = inventory.yaml
```

**Set a real admin password** (router1/router2 had none configured —
added to both `configs/router1.cfg` and `configs/router2.cfg`, not
just applied live, so it survives a redeploy):
```
username admin privilege 15 role network-admin secret admin123
```

## Bug 1 — inventory silently not loading

`ansible-inventory --list` showed only implicit localhost, nothing
under `routers`. `ansible.cfg` pointed to `inventory.yml`, the actual
file was saved as `inventory.yaml`. Ansible didn't error loudly, it
just warned and fell back to nothing parsed. Fixed by matching the
filename to what `ansible.cfg` expects (or vice versa).

Lesson: a filename mismatch here fails *silently*, not loudly — worth
always confirming `ansible-inventory --list` actually shows your hosts
before assuming the inventory file "worked."

## Bug 2 — paramiko not installed

First `eos_command` ad-hoc run:
```
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
...ConnectionError: paramiko is not installed: No module named 'paramiko'
```
Neither SSH backend `network_cli` needs was present. Fixed:
```bash
pip install paramiko
```

## Bug 3 — "Bad authentication type" (paramiko + EOS keyboard-interactive)

After installing paramiko:
```
ConnectionError: Failed to authenticate: Bad authentication type;
allowed types: ['publickey', 'keyboard-interactive']
```
Known-ish rough edge: EOS's SSH daemon often only advertises
keyboard-interactive, not plain password auth, and paramiko's handling
of that through Ansible's network_cli plugin has been flaky. Standard
recommended fix is `ansible-pylibssh` instead of paramiko:
```bash
sudo apt install -y libssh-dev python3-dev build-essential
pip install ansible-pylibssh
```

## Bug 4 — still failing with pylibssh, red herring turned out to be a stale lab

With pylibssh installed:
```
ConnectionError: ssh connection failed: Failed to authenticate public
key: Access denied for 'keyboard interactive'. Authentication that can
continue: publickey,keyboard-interactive
```
Around the same time: the Proxmox host had an earlier ungraceful power
shutdown (no UPS), and the containerlab lab state was stale. Did a full
restart. After that, manual `ssh admin@clab-clab-02-router1` with
`admin123` worked cleanly.

**Isolating the actual cause** — since two things changed at once
(restart + pylibssh installed), tested both:
1. Ran the ad-hoc `eos_command` with pylibssh installed → SUCCESS on
   both routers.
2. `pip uninstall ansible-pylibssh -y`, re-ran the exact same command
   → ALSO SUCCESS, with paramiko fallback.

**Conclusion: the stale/crashed container state from the power loss
was the actual root cause of bug 3/4, not a real paramiko/EOS
incompatibility.** Reinstalled pylibssh afterward anyway — still the
faster, generally-recommended backend for real automation work, just
not what actually fixed this particular failure.

Lesson: don't trust the first plausible-sounding fix when two things
changed at once. Isolate.

## Bug 5 — wrong eos_acls parameter name

First ACL playbook used:
```yaml
protocol_options:
  tcp:
    established: true
```
Failed:
```
Unsupported parameters for (arista.eos.eos_acls) module:
config.acls.aces.protocol_options.tcp.established. Supported
parameters include: flags.
```
Checked the actual installed module's docs rather than guessing again:
```bash
ansible-doc arista.eos.eos_acls | grep -B2 -A 15 "flags"
```
`established` is real, just nested one level deeper, under `flags`:
```yaml
protocol_options:
  tcp:
    flags:
      established: true
```

## Bug 6 — the real one: false non-idempotency from a representation mismatch

Working playbook (after bug 5's fix) pushed successfully
(`changed=1`), but repeating the exact same run kept reporting
`changed=1` again instead of `changed=0` — idempotency was broken.

`show running-config | section access-list` on the device confirmed
the ACL was correct and stable:
```
ip access-list MGMT-ACL
   10 permit tcp 172.20.20.0/24 any established
```

`ansible-playbook acl.yaml -vvv` showed the "before" and "after" facts
as visually identical to each other, yet `commands` was always:
```
ip access-list MGMT-ACL
no 10
10 permit tcp 172.20.20.0 0.0.0.255 any established
```

**Root cause**, found by comparing the raw invocation args against the
printed facts: the playbook declared the source as
```yaml
source:
  address: 172.20.20.0
  wildcard_bits: 0.0.0.255
```
but the module's facts-gathering always represents the same subnet as:
```json
"source": { "subnet_address": "172.20.20.0/24" }
```
Two different dict shapes for the same subnet. The module's
want-vs-have comparison isn't normalizing them to the same shape
before comparing, so it always concludes something changed, deletes
ACE 10, and re-adds the identical rule. Forever.

**Fix** — declare the source the same way facts represent it:
```yaml
source:
  subnet_address: "172.20.20.0/24"
```
Re-ran the three-pass proof (`--check --diff`, real apply, real apply
again) — confirmed `changed=0` on the repeat run. Genuinely idempotent
after this.

Lesson: a resource module can accept multiple valid input shapes for
the same semantic value without them being equally idempotent-safe.
When idempotency seems broken but the device state looks right, diff
the raw "before"/"after" JSON, not just the printed facts summary.

## Final working playbook

```yaml
---
- name: Push and verify a simple management ACL
  hosts: routers
  gather_facts: false
  tasks:
    - name: Ensure MGMT-ACL exists with defined rules
      arista.eos.eos_acls:
        config:
          - afi: ipv4
            acls:
              - name: MGMT-ACL
                aces:
                  - sequence: 10
                    grant: permit
                    source:
                      subnet_address: "172.20.20.0/24"
                    destination:
                      any: true
                    protocol: tcp
                    protocol_options:
                      tcp:
                        flags:
                          established: true
        state: merged
```

## Bug 7 — --check/diff mode is unreliable for eos_acls

Discovered after Bug 6 was fixed, while trying to recapture a clean
"Dry Run" section for the published post. Removed `MGMT-ACL` from
both routers, confirmed genuinely empty on both
(`show running-config | section access-list` → no output on either):

```bash
ansible-playbook acl.yaml --check --diff
```
→ `ok`/`changed=0` on both routers, despite the ACL not existing
anywhere. Then immediately:
```bash
ansible-playbook acl.yaml
```
→ `changed=1` on both, correctly created it.

**Conclusion: `--check`/`--diff` mode doesn't reflect true device
state for `arista.eos.eos_acls` — it reported "nothing to do" against
a confirmed-empty device. The real (non-check) apply path works
correctly.** This was actually visible from the very first `--check
--diff` run all the way back at Bug 6 too (it also said changed=0
before anything had ever been pushed) — at the time that got
attributed to the idempotency bug being investigated, not recognized
as a separate, standing check-mode issue until retested against a
verified-clean state.

Lesson: don't trust `--check --diff` output for this module as a
preview of what will happen. Verify real device state directly
instead, and treat the real apply's `changed` result as the source of
truth, not the dry run's.
