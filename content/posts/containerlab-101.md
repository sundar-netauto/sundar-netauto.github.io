---
title: "Containerlab 101: Basics"
date: 2026-09-01
draft: false
tags:
  - containerlab
  - docker
  - arista
  - network-labs
  - homelab
summary: What containerlab actually is, the handful of commands that matter (deploy/inspect/destroy), how topology files and node 'kinds' work.
---
> Containerlab launches, wires up and manages container-based labs

## Lab 1: A Single Router

Below is a simple `topology definition file` . When deployed it spins up an Arista CEOS router.

```bash
devops@netdevops-lab-vm02:~/clab$ cat lab01.clab.yaml 
name: lab01

topology:
  nodes:
    ceos1:
      kind: arista_ceos
      image: ceos:4.36.2F
```

> Try removing the clab from the file extension and see what happens. 

```bash
devops@netdevops-lab-vm02:~/clab$ sudo containerlab deploy lab01.clab.yaml 
02:22:41 INFO Containerlab started version=0.79.0
02:22:41 INFO Parsing & checking topology file=lab01.clab.yaml
02:22:41 INFO Creating docker network name=clab IPv4 subnet=172.20.20.0/24 IPv6 subnet=3fff:172:20:20::/64 MTU=0
02:22:41 INFO Creating lab directory path=/home/devops/clab/clab-lab01
02:22:41 INFO Creating container name=ceos1
02:22:42 INFO Running postdeploy actions for Arista cEOS 'ceos1' node
02:23:16 INFO Adding host entries path=/etc/hosts
02:23:16 INFO Adding SSH config for nodes path=/etc/ssh/ssh_config.d/clab-lab01.conf
╭──────────────────┬──────────────┬─────────┬───────────────────╮
│       Name       │  Kind/Image  │  State  │   IPv4/6 Address  │
├──────────────────┼──────────────┼─────────┼───────────────────┤
│ clab-lab01-ceos1 │ arista_ceos  │ running │ 172.20.20.2       │
│                  │ ceos:4.36.2F │         │ 3fff:172:20:20::2 │
╰──────────────────┴──────────────┴─────────┴───────────────────╯
```

>Note the node name of the running container to the name you defined in the yaml. Do you see any difference ? 

> Check your directory to see if you notice any new folders created by containerlab. Inspect the folder. 

### Connecting to router

```bash
devops@netdevops-lab-vm02:~/clab$ docker exec -it clab-lab01-ceos1 Cli
ceos1>en
ceos1#show version | grep version
Hardware version: 
Software image version: 4.36.2F-49692632.4362F.1 (engineering build)
Internal build version: 4.36.2F-49692632.4362F.1
Image format version: 1.0
Kernel version: 5.15.0-190-generic
ceos1#
```

```bash
devops@netdevops-lab-vm02:~/clab$ ssh admin@172.20.20.2
(admin@172.20.20.2) Password: 
ceos1>en
ceos1#show clock
Tue Sep 01 02:29:44 2026
Timezone: UTC
Clock source: local
ceos1#

```


```
devops@netdevops-lab-vm02:~/clab$ docker exec clab-lab01-ceos1 FastCli -p 15 -c "show version | grep version"
Hardware version: 
Software image version: 4.36.2F-49692632.4362F.1 (engineering build)
Internal build version: 4.36.2F-49692632.4362F.1
Image format version: 1.0
Kernel version: 5.15.0-190-generic

```



### Tearing It Down

```bash
devops@netdevops-lab-vm02:~/clab$ sudo containerlab destroy 
02:38:41 INFO Using topology file file=lab01.clab.yaml
02:38:41 INFO Parsing & checking topology file=lab01.clab.yaml
02:38:41 INFO Parsing & checking topology file=lab01.clab.yaml
02:38:41 INFO Destroying lab name=lab01
02:38:42 INFO Removed container name=clab-lab01-ceos1
02:38:42 INFO Removing host entries path=/etc/hosts
02:38:42 INFO Removing SSH config path=/etc/ssh/ssh_config.d/clab-lab01.conf
```


## Lab 2: Mixing Kinds

Here is another topology
```bash
devops@netdevops-lab-vm02:~/clab02$ cat clab02.clab.yaml 
name: clab-02

topology:
  nodes:
    router1:
      kind: arista_ceos
      image: ceos:4.36.2F
      startup-config: configs/router1.cfg

    router2:
      kind: arista_ceos
      image: ceos:4.36.2F
      startup-config: configs/router2.cfg

    host1:
      kind: linux
      image: alpine:latest
      exec:
        - ip addr add 10.10.10.11/24 dev eth1
        - ip link set eth1 up


  links:
    # router1 <-> router2
    - endpoints: ["router1:eth1", "router2:eth1"]
    # hosts
    - endpoints: ["router1:eth2", "host1:eth1"]

```

We have three nodes here , router1 and router2.  They are of kind arista_ceos. Look at the third node, it's of kind linux . 

> Kind abstracts away the need to understand certain setup peculiarities of different Network Operating Systems. For the list of supported kinds refer to https://containerlab.dev/manual/kinds/ As a user , check if the NOS you are using is supported and call the kind. 

Both routers are connected via their eth1 as defined under links section of this topology definition file. 

You also see a startup-config field which lets you load a configuration from a file. 

```bash
devops@netdevops-lab-vm02:~/clab02$ tree
.
├── clab02.clab.yaml
└── configs
    ├── router1.cfg
    └── router2.cfg

1 directory, 3 files
devops@netdevops-lab-vm02:~/clab02$ cat configs/router1.cfg
hostname router1
!
ip routing
!
!
interface Ethernet1
   no switchport
   ip address 10.1.1.0/31
!
interface Ethernet2
   no switchport
   ip address 10.10.10.254/24
!
end

```

All the way on the top is the name of this lab.  In hindsight, what you looked at is a `topology definition file` , it has .clab.yaml extension. It contains the topology which is a set of nodes and links. 

### Deploying It

Tired of typing the long commands, lets setup alias 

`alias clab="sudo containerlab"`  

Now we will run containerlab with the alias this time for the second lab. 

```
devops@netdevops-lab-vm02:~/clab02$ clab deploy clab02.clab.yaml 
04:15:23 INFO Containerlab started version=0.79.0
04:15:23 INFO Parsing & checking topology file=clab02.clab.yaml
04:15:23 INFO Creating lab directory path=/home/devops/clab02/clab-clab-02
04:15:23 INFO Creating container name=host1
04:15:23 INFO Creating container name=router2
04:15:23 INFO Creating container name=router1
04:15:23 INFO Running postdeploy actions for Arista cEOS 'router2' node
04:15:24 INFO Created link: router1:eth1 ▪┄┄▪ router2:eth1
04:15:24 INFO Created link: router1:eth2 ▪┄┄▪ host1:eth1
04:15:24 INFO Running postdeploy actions for Arista cEOS 'router1' node
04:16:29 INFO Executed command node=host1 command="ip addr add 10.10.10.11/24 dev eth1" stdout=""
04:16:29 INFO Executed command node=host1 command="ip link set eth1 up" stdout=""
04:16:29 INFO Adding host entries path=/etc/hosts
04:16:29 INFO Adding SSH config for nodes path=/etc/ssh/ssh_config.d/clab-clab-02.conf
╭──────────────────────┬───────────────┬─────────┬───────────────────╮
│         Name         │   Kind/Image  │  State  │   IPv4/6 Address  │
├──────────────────────┼───────────────┼─────────┼───────────────────┤
│ clab-clab-02-host1   │ linux         │ running │ 172.20.20.2       │
│                      │ alpine:latest │         │ 3fff:172:20:20::2 │
├──────────────────────┼───────────────┼─────────┼───────────────────┤
│ clab-clab-02-router1 │ arista_ceos   │ running │ 172.20.20.4       │
│                      │ ceos:4.36.2F  │         │ 3fff:172:20:20::4 │
├──────────────────────┼───────────────┼─────────┼───────────────────┤
│ clab-clab-02-router2 │ arista_ceos   │ running │ 172.20.20.3       │
│                      │ ceos:4.36.2F  │         │ 3fff:172:20:20::3 │
╰──────────────────────┴───────────────┴─────────┴───────────────────╯

```


> Note the deploy logs , do you see link creation ? 


Let's try few more alias to make things a bit easier
```
alias clab-host1="docker exec clab-clab-02-host1 sh -c"
alias clab-rtr1="docker exec clab-clab-02-router1 FastCli -p 15 -c"
alias clab-rtr2="docker exec clab-clab-02-router2 FastCli -p 15 -c"
```

```bash
devops@netdevops-lab-vm02:~/clab02$ clab-host1 "uname -a"
Linux host1 5.15.0-190-generic #200-Ubuntu SMP Fri Aug 7 15:06:04 UTC 2026 x86_64 Linux
devops@netdevops-lab-vm02:~/clab02$ clab-rtr1 "show hostname"
Hostname: router1
FQDN:     router1
devops@netdevops-lab-vm02:~/clab02$ clab-rtr2 "show version | grep version"
Hardware version: 
Software image version: 4.36.2F-49692632.4362F.1 (engineering build)
Internal build version: 4.36.2F-49692632.4362F.1
Image format version: 1.0
Kernel version: 5.15.0-190-generic
```

### Verifying Connectivity

Let's check how the startup config we loaded look like.

```
devops@netdevops-lab-vm02:~/clab02$ clab-rtr1 "show ip int bri | i Ethernet1"
Ethernet1       10.1.1.0/31         up          up              1500           
devops@netdevops-lab-vm02:~/clab02$ clab-rtr2 "show ip int bri | i Ethernet1"
Ethernet1       10.1.1.1/31         up          up              1500           
devops@netdevops-lab-vm02:~/clab02$ clab-rtr2 "ping  10.1.1.0"
PING 10.1.1.0 (10.1.1.0) 72(100) bytes of data.
80 bytes from 10.1.1.0: icmp_seq=1 ttl=64 time=0.099 ms
80 bytes from 10.1.1.0: icmp_seq=2 ttl=64 time=0.026 ms
80 bytes from 10.1.1.0: icmp_seq=3 ttl=64 time=0.026 ms
80 bytes from 10.1.1.0: icmp_seq=4 ttl=64 time=0.034 ms
80 bytes from 10.1.1.0: icmp_seq=5 ttl=64 time=0.029 ms

--- 10.1.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.026/0.042/0.099/0.028 ms, ipg/ewma 0.114/0.070 ms

devops@netdevops-lab-vm02:~/clab02$ clab-host1 "ping -c 4 10.10.10.254"
PING 10.10.10.254 (10.10.10.254): 56 data bytes
64 bytes from 10.10.10.254: seq=0 ttl=64 time=0.126 ms
64 bytes from 10.10.10.254: seq=1 ttl=64 time=0.119 ms
64 bytes from 10.10.10.254: seq=2 ttl=64 time=0.114 ms
64 bytes from 10.10.10.254: seq=3 ttl=64 time=0.131 ms

--- 10.10.10.254 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.114/0.122/0.131 ms

```



If you want to follow what's described above, ensure containerlab is installed and the arista docker image is downloaded. 
## Installation

```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

```
devops@netdevops-lab-vm02:~/clab$ docker image ls | grep ceos
ceos:4.36.2F         b63b623c95f3       2.45GB             0B    
```

## Command Summary

```
sudo containerlab deploy <topo-def.clab.yaml>
sudo containerlab inspect <topo-def.clab.yaml>
sudo containerlab destroy <topo-def.clab.yaml>
```

These are the three common commands , they are pretty much self explanatory.

### Connectivity

```
ssh <user>@<ip-addr-of-node>
docker exec <clab-node-name> FastCli -p 15 -c "<command>"
docker exec -it <clab-node-name> Cli
```

You have multiple ways to connect to the running node.
### Alias

```
alias clab-host1='docker exec clab-clab-02-host1 sh -c'
alias clab-rtr1='docker exec clab-clab-02-router1 FastCli -p 15 -c'
alias clab-rtr2='docker exec clab-clab-02-router2 FastCli -p 15 -c'

```





