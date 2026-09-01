---
title: "Ansible 101: Basics"
date: 2026-09-01
draft: false
tags:
  - ansible
  - containerlab
  - arista
  - network-automation
  - homelab
summary: What is ansible. How to setup and use.
---

> Ansible is an agentless configuration management tool, used to configure routers, servers or anything which lets you connect over SSH or an API.


> Building Blocks

> Playbook - Yaml file with a set of instructions (Tasks) on what to do.

> Inventory - .ini or .yaml file with the list of nodes and its credentials.

> Connection Plugin - The transport mechanism to connect to the remote node, e.g. network_cli, ssh, httpapi, with underlying transport libraries such as Paramiko, pylibssh.

> Modules - Network OS (e.g. ios, nxos, ceos) specific actions.

> Collections - A bundle of modules, plugins and roles for a specific vendor/platform (e.g. arista.eos).

## Prerequisites



```bash
sudo apt update
sudo apt install -y python3-venv python3-pip
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
```

```bash
pip install ansible
ansible --version

sudo apt install -y libssh-dev python3-dev build-essential
pip install ansible-pylibssh # default choice, else will fallback to paramiko
# pip install paramiko # you need either one of this or pylibssh.
```

```
(ansible-venv) devops@netdevops-lab-vm02:~$ ansible --version
ansible [core 2.17.14]
  config file = None
  configured module search path = ['/home/devops/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/devops/ansible-venv/lib/python3.10/site-packages/ansible
  ansible collection location = /home/devops/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/devops/ansible-venv/bin/ansible
  python version = 3.10.12 (main, Jul 15 2026, 23:40:17) [GCC 11.4.0] (/home/devops/ansible-venv/bin/python3)
  jinja version = 3.1.6
  libyaml = True

```

If the plugin is not installed by default, install it manually using the command below. To check, use `ansible-galaxy collection list | grep -E "arista|netcommon"`
```bash
ansible-galaxy collection install arista.eos
```
## Inventory


```bash
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ tree
.
├── ansible.cfg
└── inventory.yaml

0 directories, 2 files

```

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

```
[defaults]
host_key_checking = False
inventory = inventory.yaml
```

```
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ ansible-inventory --list
{
    "_meta": {
        "hostvars": {
            "router1": {
                "ansible_become": true,
                "ansible_become_method": "enable",
                "ansible_connection": "ansible.netcommon.network_cli",
                "ansible_host": "clab-clab-02-router1",
                "ansible_network_os": "arista.eos.eos",
                "ansible_password": "admin123",
                "ansible_user": "admin"
            },
            "router2": {
                "ansible_become": true,
                "ansible_become_method": "enable",
                "ansible_connection": "ansible.netcommon.network_cli",
                "ansible_host": "clab-clab-02-router2",
                "ansible_network_os": "arista.eos.eos",
                "ansible_password": "admin123",
                "ansible_user": "admin"
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "routers"
        ]
    },
    "routers": {
        "hosts": [
            "router1",
            "router2"
        ]
    }
}

```
## First Contact: An Ad-hoc Command



```bash
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ ansible routers -m arista.eos.eos_command -a "commands='show version'"
router1 | SUCCESS => {
    "changed": false,
    "stdout": [
        "Arista cEOSLab\nHardware version: \nSerial number: 42FCA92E9449B68DF7C3E07EC192E3CE\nHardware MAC address: 001c.736d.4377\nSystem MAC address: 001c.736d.4377\n\nSoftware image version: 4.36.2F-49692632.4362F.1 (engineering build)\nArchitecture: i686\nInternal build version: 4.36.2F-49692632.4362F.1\nInternal build ID: 4c2c31a5-28af-417f-b85e-464a08c0a717\nImage format version: 1.0\nImage optimization: None\n\nKernel version: 5.15.0-190-generic\n\nUptime: 28 minutes\nTotal memory: 8127804 kB\nFree memory: 5644632 kB"
    ],
    "stdout_lines": [
        [
            "Arista cEOSLab",
            "Hardware version: ",
            "Serial number: 42FCA92E9449B68DF7C3E07EC192E3CE",
            "Hardware MAC address: 001c.736d.4377",
            "System MAC address: 001c.736d.4377",
            "",
            "Software image version: 4.36.2F-49692632.4362F.1 (engineering build)",
            "Architecture: i686",
            "Internal build version: 4.36.2F-49692632.4362F.1",
            "Internal build ID: 4c2c31a5-28af-417f-b85e-464a08c0a717",
            "Image format version: 1.0",
            "Image optimization: None",
            "",
            "Kernel version: 5.15.0-190-generic",
            "",
            "Uptime: 28 minutes",
            "Total memory: 8127804 kB",
            "Free memory: 5644632 kB"
        ]
    ]
}
router2 | SUCCESS => {
    "changed": false,
    "stdout": [
        "Arista cEOSLab\nHardware version: \nSerial number: 3F91CB167E61E757357D6FC4659D9B43\nHardware MAC address: 001c.73cc.7980\nSystem MAC address: 001c.73cc.7980\n\nSoftware image version: 4.36.2F-49692632.4362F.1 (engineering build)\nArchitecture: i686\nInternal build version: 4.36.2F-49692632.4362F.1\nInternal build ID: 4c2c31a5-28af-417f-b85e-464a08c0a717\nImage format version: 1.0\nImage optimization: None\n\nKernel version: 5.15.0-190-generic\n\nUptime: 28 minutes\nTotal memory: 8127804 kB\nFree memory: 5644632 kB"
    ],
    "stdout_lines": [
        [
            "Arista cEOSLab",
            "Hardware version: ",
            "Serial number: 3F91CB167E61E757357D6FC4659D9B43",
            "Hardware MAC address: 001c.73cc.7980",
            "System MAC address: 001c.73cc.7980",
            "",
            "Software image version: 4.36.2F-49692632.4362F.1 (engineering build)",
            "Architecture: i686",
            "Internal build version: 4.36.2F-49692632.4362F.1",
            "Internal build ID: 4c2c31a5-28af-417f-b85e-464a08c0a717",
            "Image format version: 1.0",
            "Image optimization: None",
            "",
            "Kernel version: 5.15.0-190-generic",
            "",
            "Uptime: 28 minutes",
            "Total memory: 8127804 kB",
            "Free memory: 5644632 kB"
        ]
    ]
}

```

## The Playbook: A Simple ACL



```yaml
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ cat acl.yaml 
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
                      subnet_address: 172.20.20.0/24
                    destination:
                      any: true
                    protocol: tcp
                    protocol_options:
                      tcp:
                        flags:
                          established: true
        state: merged

```


### Applying It



```bash
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ ansible-playbook acl.yaml

PLAY [Push and verify a simple management ACL] ******************************************************************************************************************************************************************************************************************************

TASK [Ensure MGMT-ACL exists with defined rules] ****************************************************************************************************************************************************************************************************************************
changed: [router2]
changed: [router1]

PLAY RECAP ******************************************************************************************************************************************************************************************************************************************************************
router1                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
router2                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```


## Verifying It Landed


```bash
(ansible-venv) devops@netdevops-lab-vm02:~/ansible101$ clab-rtr1 "show running-config | section access-list"
> show running-config | section access-list
ip access-list MGMT-ACL
   10 permit tcp 172.20.20.0/24 any established

```


## Command Summary



**Collection**
```bash
ansible-galaxy collection install arista.eos
ansible-galaxy collection list | grep -E "arista|netcommon"
```

**Inventory**
```bash
ansible-inventory --list
```

**Ad-hoc command**
```bash
ansible routers -m arista.eos.eos_command -a "commands='show version'"
```

**Run a playbook**
```bash
ansible-playbook acl.yaml
```

