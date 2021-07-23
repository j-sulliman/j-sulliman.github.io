---
layout: default
title: 1. Deploying an EVPN and VXLAN Fabric with Ansible
parent: Infrastructure as Code
nav_order: 1
---

# Introduction


Custom scripting and automation works well in the project space where there's usually one or two delivery engineers  implementing the solution.  But what happens when the configuration automation tools need to be used, supported and maintained by a wider team on an ongoing basis?  

Is the team equipped to maintain often poorly documented python code?  Who's going to support and maintain that code if you leave?  Is it secure?  Free of bugs?

The further we progress from delivery - design, implementation to operational hand-over, the more important the choice and supportability of automation tools becomes.

This is where configuration automation or "Infrastructure as Code" tools such as Ansible have greater relevance, automation of environments at scale, where supportability and re-use by a wider team of engineers, each likely with varying levels of programming and automation skills.... "DevOps" environments.

In a [previous article](https://j-sulliman.github.io/2021/05/04/Part-3.1-DCNM-Lab-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) we stepped through creating a virtualised EVPN and VXLAN lab using DCNM and N9000vs. This article, I outline my learnings from my first real foray into Ansible (better late than never!) - using playbooks and templates to deploy a non DCNM controlled fabric.  

##### Ansible and EVPN VXLAN Deployment - adjust the playback speed to suit.

<iframe width="700" height="400" src="https://www.youtube.com/embed/OPMPnn5aJHY" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

> Note: 9 of 10 times you will likely want to use a controller managed fabric solution - (APICs for ACI, DCNM for standalone) to automate, provision, manage your environment. The intent of this article is to document and share my learnings with Ansible and NXOS.  


# References and Resources

All of the ansible files, hosts, playbooks variables, used in this project can be found in the public github repository [ansible_nxos_evpn](https://github.com/j-sulliman/ansible_nxos_evpn.git).

Other resources I found really useful:
* The free introductory Ansible course available from [RedHat](https://www.redhat.com/en/blog/new-free-ansible-course).
* The structured labs and learning available for Ansible that can be found on [DevNet](https://developer.cisco.com/automation-ansible/).  
* [LTRDCN-1572 - Ondemand Learning Library](https://www.ciscolive.com/global/on-demand-library.html?search=ansible%20VXLAN#/session/1532112857665001tt4c).  An excellent lab that forms the basis of the playbooks used here


# Lab Overview

##### High Level Topology

![ACI Dashboard](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Anisble_EVPN_VXLAN.png?raw=true)


#### Virtual Appliances

Appliance | Image name | Version | Memory | CPU | Disk | Qty
------------ | ------------- | ------------- | ------------- | ------------- | ------------- | -------------
n9000v | nexus9300v.9.3.7.qcow2 | 9.3.7 | 8096MB | 2vCPUs | - | 6
Ubuntu Desktop | Ubuntu 20.10 (64bit).vmdk | 20.10 | 1024MB | 1vCPUs | - | 3



#### Device Overview

Hostname | MGT Addr | Loopback0 | Loopback1 | Role
------------ | ------------- | ------------- | -------------
dc1-leaf-001 | 192.168.200.113 | 10.1.1.1/32 | 10.1.1.111/32 | VTEP, BGP RR Client, Access
dc1-leaf-002 | 192.168.200.114 | 10.1.1.2/32 | 10.1.1.112/32 | VTEP, BGP RR Client, Access
dc1-leaf-003 | 192.168.200.115 | 10.1.1.3/32 | 10.1.1.113/32 | VTEP, BGP RR Client, Access
dc1-leaf-004 | 192.168.200.116 | 10.1.1.4/32 | 10.1.1.114/32 | VTEP, BGP RR Client, Access
dc1-spine-001 | 192.168.200.117 | 10.1.1.101/32 | 10.1.1.200/32 | BGP RR, PIM RP
dc1-spine-001 | 192.168.200.118 | 10.1.1.102/32| 10.1.1.200/32  | BGP RR, PIM RP

#### Underlay Interfaces - Leafs

Hostname | Interface | Description | IP Address
------------ | ------------- | -------------
dc1-leaf-001 | Ethernet1/1 | dc1-spine-001_E1/1 | 10.1.10.1/30
dc1-leaf-001 | Ethernet1/2 | dc1-spine-002_E1/1 | 10.1.10.5/30
dc1-leaf-002 | Ethernet1/1 | dc1-spine-001_E1/2 | 10.1.10.9/30
dc1-leaf-002 | Ethernet1/2 | dc1-spine-002_E1/2 | 10.1.10.13/30
dc1-leaf-003 | Ethernet1/1 | dc1-spine-001_E1/3 | 10.1.10.17/30
dc1-leaf-003 | Ethernet1/2 | dc1-spine-002_E1/3 | 10.1.10.21/30
dc1-leaf-004 | Ethernet1/1 | dc1-spine-001_E1/4 | 10.1.10.25/30
dc1-leaf-004 | Ethernet1/2 | dc1-spine-002_E1/4 | 10.1.10.29/30


#### Underlay Interfaces - Spines

Hostname | Interface | Description | IP Address
------------ | ------------- | -------------
dc1-spine-001 | Ethernet1/1 | dc1-leaf-001_E1/1  | 10.1.10.2/30
dc1-spine-001 | Ethernet1/2 | dc1-leaf-002_E1/1  | 10.1.10.10/30
dc1-spine-001 | Ethernet1/3 | dc1-leaf-003_E1/1  | 10.1.10.18/30
dc1-spine-001 | Ethernet1/4 | dc1-leaf-004_E1/1  | 10.1.10.26/30
dc1-spine-002 | Ethernet1/1 | dc1-leaf-001_E1/2  | 10.1.10.6/30
dc1-spine-002 | Ethernet1/2 | dc1-leaf-002_E1/2  | 10.1.10.14/30
dc1-spine-002 | Ethernet1/3 | dc1-leaf-003_E1/2  | 10.1.10.22/30
dc1-spine-002 | Ethernet1/4 | dc1-leaf-004_E1/2  | 10.1.10.30/30


# Ansible Overview and Setup

An overview of some of the ansible terminology I'll refer to through this article:

* **Tasks**: A command executed against a host.
* **Playbooks**: A sequence of commands executed against a host. We'll use eight playbooks for our EVPN and VXLAN deployment.
* **Variables**:  Configuration parameters that are unique to each host (e.g. Loopback0 address) are defined in variables, which can be defined in multiple places.  The playbooks we'll use reference variables in the hosts (inventory file), host_vars and group_vars directories - see the directory structure below for an example.  You can read more on variables in the [User Guide](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#ansible-variable-precedence)
* **Hosts File**:  An inventory of devices that playbooks will be run against is defined in the **hosts** file.  Our hosts file will include two groups [spines] and [leafs].  Within the hosts file you can also define aliases and host vars.  Loopback addresses and hostnames for example.  Defining host specific variables in the hosts file soon becomes unwieldily to manage as the number of variables grow.  We'll look at some alternatives - group_vars and host_vars below.
* **Collections**: The [Cisco Nxos collection](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/index.html#plugins-in-cisco-nxos) will be used throughout our playbooks


#### Pre-requisites

* Before running the playbooks, from console boot the NXOS image `loader > boot nxos.9.3.7.bin` and assign the management addresses as defined in your hosts file.
* Install ansible - in my case `pip install ansible` inside a virtual environment.  See the [Install Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
*  A working knowledge of EVPN and VXLAN on Nexus switches - recommend reading the [design guide](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-9000-series-switches/guide-c07-734107.html) for fundamentals

An outline of the files and directory structure used for this project is shown below.

##### Project and Directory Structure
```
.
├── 1_global_configuration.yml
├── 2_configure_spines.yml
├── 3_configure_leafs.yml
├── 4_configure_bgp_evpn_spines.yml
├── 5_configure_bgp_evpn_leafs.yml
├── 6_configure_vxlan_leafs_tenant_1.yml
├── 7_configure_vxlan_leafs_tenant_2.yml
├── 8_verify_fabric.yml
├── Ansible_snapshot
├── ansible.cfg
├── configure_routes.yml
├── first_playbook.yml
├── group_vars
│   ├── leafs.yml
│   └── spines.yml
├── host_vars
│   ├── dc1-leaf-001
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-002
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-003
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-004
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-spine-001
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   └── dc1-spine-002
│       ├── interfaces.yml
│       └── vpc.yml
├── hosts
```

## Ansible configuration file
Ansible configuration file is as default except for:
```
[defaults]
inventory = hosts
host_key_checking = false
record_host_key = true
stdout_callback = debug
ansible_python_interpreter=venv/bin/python3.9
```

## Inventory and Hosts file
Create groups within your hosts file that makes sense for your environment.  In the case of our lab:
* Groups are defined in [brackets], we define Spine, Leafs  as `[leafs] [spines]`
* Under each group we define the Management address and hostname of member switches
* We define group specific variables (username and password) using the `[group:vars]` notation


Here's the complete hosts file we'll use for our lab.  Edit the management addresses and credentials as needed.
```
[dc1:children]
spines
leafs
[dc1:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=nxos
ansible_user=admin
ansible_password=Cisco123!
ansible_python_interpreter=venv/bin/python3.9

[leafs]
dc1-leaf-001 ansible_host=192.168.200.113 hostname=dc1-leaf-001
dc1-leaf-002 ansible_host=192.168.200.114 hostname=dc1-leaf-002
dc1-leaf-003 ansible_host=192.168.200.115 hostname=dc1-leaf-003
dc1-leaf-004 ansible_host=192.168.200.116 hostname=dc1-leaf-004

[leafs:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=nxos
ansible_user=admin
ansible_password=Cisco123!
ansible_python_interpreter=venv/bin/python3.9


[spines]
dc1-spine-001 ansible_host=192.168.200.117 hostname=dc1-spine-001 loopback=10.1.1.101/32
dc1-spine-002 ansible_host=192.168.200.118 hostname=dc1-spine-002 loopback=10.1.1.102/32

[spines:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=nxos
ansible_user=admin
ansible_password=Cisco123!
ansible_python_interpreter=venv/bin/python3.9
```

## Variables

Variables define configuration items that are unique to each switch, loopback0 and underlay addresses for example.

Variables can be defined in multiple places, from the command line, role defaults, the inventory (hosts) file, group_vars and host_vars and from the playbook itself.  

This also represents the high-level precedence that variables will be applied, for example a loopback0 address defined in the hosts file is preferred over a lopback0 variable defined in a host_vars folder.

To keep variable definitions manageable, use of hosts file for variables will be kept to a minimum. Instead we'll define variables common to a group of switches in the `group_vars\'group-name'.yml` files and variables unique to each switch are defined under the `host_vars\ 'hostname'\'variable-file'.yml `

Below shows the structure of files that will contain the variables for this deployment.

Refer to  Ansible Variable Documentation [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#playbooks-variables)

```
├── group_vars
│   ├── leafs.yml
│   └── spines.yml
├── host_vars
│   ├── dc1-leaf-001
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-002
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-003
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-leaf-004
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   ├── dc1-spine-001
│   │   ├── interfaces.yml
│   │   └── vpc.yml
│   └── dc1-spine-002
│       ├── interfaces.yml
│       └── vpc.yml
├── hosts
```



### Inventory or host file vars

Only Management address and hostnames are defined here.

### Host vars
The host_vars folder will contain a folder for each device, the folder name is the same as the hostname defined in our hosts file.  Interface addressing, router-ids and spine rp-addresses are defined here.

Sample  **interfaces.yml** hosts_vars file for **Leafs**:
```yaml
loopback0: 10.1.1.1/32
loopback1: 10.1.1.111/32
router_id: 10.1.1.1
eth1: 10.1.10.1/30
eth2: 10.1.10.5/30
access_vlan: 10

L3_interfaces:
- { interface: Ethernet1/1, descr: Underlay_dc1-spine-001_Ethernet1/1 }
- { interface: Ethernet1/2, descr: Underlay_dc1-spine-002_Ethernet1/1 }

L3_loopbacks:
- { interface: loopback0, descr: RID and Update Source }
- { interface: loopback1, descr: Anycast RP }

Underlay_interfaces:
  - { underlay_int: Ethernet1/1, underlay_addr: 10.1.10.1/30, underlay_descr: Underlay_dc1-spine-001_Ethernet1/1 }
  - { underlay_int: Ethernet1/2, underlay_addr: 10.1.10.5/30, underlay_descr: Underlay_dc1-spine-002_Ethernet1/1 }

Lag_interfaces:
  - { lag_name: port-channel1, lag_member: Ethernet1/3, lag_mode: active}

rp_address: 10.1.1.200
s1_loopback: 10.1.1.101
s2_loopback: 10.1.1.102
```

Sample  **interfaces.yml** hosts_vars file for **Spines**:
```yaml
loopback0: 10.1.1.101/32
loopback1: 10.1.1.200/32
router_id: 10.1.1.101
eth1: 10.1.10.2/30
eth2: 10.1.10.10/30
eth3: 10.1.10.18/30
eth4: 10.1.10.26/30


L3_interfaces:
- { interface: Ethernet1/1, descr: Underlay_dc1-leaf-001_Ethernet1/1 }
- { interface: Ethernet1/2, descr: Underlay_dc1-leaf-002_Ethernet1/1 }
- { interface: Ethernet1/3, descr: Underlay_dc1-leaf-003_Ethernet1/1 }
- { interface: Ethernet1/4, descr: Underlay_dc1-leaf-004_Ethernet1/1 }

L3_loopbacks:
- { interface: loopback0, descr: RID and Update Source }
- { interface: loopback1, descr: Anycast RP }

Underlay_interfaces:
  - { underlay_int: Ethernet1/1, underlay_addr: 10.1.10.2/30, underlay_descr: Underlay_dc1-leaf-001_Ethernet1/1 }
  - { underlay_int: Ethernet1/2, underlay_addr: 10.1.10.10/30, underlay_descr: Underlay_dc1-leaf-002_Ethernet1/1 }
  - { underlay_int: Ethernet1/3, underlay_addr: 10.1.10.18/30, underlay_descr: Underlay_dc1-leaf-003_Ethernet1/1 }
  - { underlay_int: Ethernet1/4, underlay_addr: 10.1.10.26/30, underlay_descr: Underlay_dc1-leaf-004_Ethernet1/1 }

loopback1_rp: 10.1.1.200
s1_loopback: 10.1.1.101
s2_loopback: 10.1.1.102
```

### Group vars

The group_vars contains configuration common to all leaves.  For our deployment, we define the BGP AS and nieghbours, tenant VLAN IDs and VNIDs, SVI addresses and multicast groups here.

**group_vars/leafs.yml** file:
 ```yaml
 asn: 65000

bgp_neighbors:
  - { remote_as: 65000, neighbor: 10.1.1.101, update_source: Loopback0 }
  - { remote_as: 65000, neighbor: 10.1.1.102, update_source: Loopback0 }

L2VNI_Tenant1:
  - { vlan_id: 10, vni: 50010, ip_add: 10.1.10.1/24, vlan_name: L2-VNI-10-Tenant1, mcast: 239.0.0.10 }
  - { vlan_id: 11, vni: 50011, ip_add: 10.1.11.1/24, vlan_name: L2-VNI-11-Tenant1, mcast: 239.0.0.11 }
  - { vlan_id: 12, vni: 50012, ip_add: 10.1.12.1/24, vlan_name: L2-VNI-12-Tenant1, mcast: 239.0.0.12 }
  - { vlan_id: 13, vni: 50013, ip_add: 10.1.13.1/24, vlan_name: L2-VNI-13-Tenant1, mcast: 239.0.0.13 }
  - { vlan_id: 14, vni: 50014, ip_add: 10.1.14.1/24, vlan_name: L2-VNI-14-Tenant1, mcast: 239.0.0.14 }
  - { vlan_id: 15, vni: 50015, ip_add: 10.1.15.1/24, vlan_name: L2-VNI-15-Tenant1, mcast: 239.0.0.15 }
  - { vlan_id: 16, vni: 50016, ip_add: 10.1.16.1/24, vlan_name: L2-VNI-16-Tenant1, mcast: 239.0.0.16 }
  - { vlan_id: 17, vni: 50017, ip_add: 10.1.17.1/24, vlan_name: L2-VNI-17-Tenant1, mcast: 239.0.0.17 }
  - { vlan_id: 18, vni: 50018, ip_add: 10.1.18.1/24, vlan_name: L2-VNI-18-Tenant1, mcast: 239.0.0.18 }
  - { vlan_id: 19, vni: 50019, ip_add: 10.1.19.1/24, vlan_name: L2-VNI-19-Tenant1, mcast: 239.0.0.19 }
  - { vlan_id: 20, vni: 50020, ip_add: 10.1.20.1/24, vlan_name: L2-VNI-20-Tenant1, mcast: 239.0.0.20 }
  - { vlan_id: 21, vni: 50021, ip_add: 10.1.21.1/24, vlan_name: L2-VNI-21-Tenant1, mcast: 239.0.0.21 }
  - { vlan_id: 22, vni: 50022, ip_add: 10.1.22.1/24, vlan_name: L2-VNI-22-Tenant1, mcast: 239.0.0.22 }
  - { vlan_id: 23, vni: 50023, ip_add: 10.1.23.1/24, vlan_name: L2-VNI-23-Tenant1, mcast: 239.0.0.23 }

L2VNI_Tenant2:
  - { vlan_id: 110, vni: 50110, ip_add: 10.1.110.1/24, vlan_name: L2-VNI-110-Tenant2, mcast: 239.0.0.110 }
  - { vlan_id: 111, vni: 50111, ip_add: 10.1.111.1/24, vlan_name: L2-VNI-111-Tenant2, mcast: 239.0.0.111 }
  - { vlan_id: 112, vni: 50112, ip_add: 10.1.112.1/24, vlan_name: L2-VNI-112-Tenant2, mcast: 239.0.0.112 }
  - { vlan_id: 113, vni: 50113, ip_add: 10.1.113.1/24, vlan_name: L2-VNI-113-Tenant2, mcast: 239.0.0.113 }
  - { vlan_id: 114, vni: 50114, ip_add: 10.1.114.1/24, vlan_name: L2-VNI-114-Tenant2, mcast: 239.0.0.114 }
  - { vlan_id: 115, vni: 50115, ip_add: 10.1.115.1/24, vlan_name: L2-VNI-115-Tenant2, mcast: 239.0.0.115 }
  - { vlan_id: 116, vni: 50116, ip_add: 10.1.116.1/24, vlan_name: L2-VNI-116-Tenant2, mcast: 239.0.0.116 }
  - { vlan_id: 117, vni: 50117, ip_add: 10.1.117.1/24, vlan_name: L2-VNI-117-Tenant2, mcast: 239.0.0.117 }
  - { vlan_id: 118, vni: 50118, ip_add: 10.1.118.1/24, vlan_name: L2-VNI-118-Tenant2, mcast: 239.0.0.118 }
  - { vlan_id: 119, vni: 50119, ip_add: 10.1.119.1/24, vlan_name: L2-VNI-119-Tenant2, mcast: 239.0.0.119 }
  - { vlan_id: 120, vni: 50120, ip_add: 10.1.120.1/24, vlan_name: L2-VNI-120-Tenant2, mcast: 239.0.0.120 }
  - { vlan_id: 121, vni: 50121, ip_add: 10.1.121.1/24, vlan_name: L2-VNI-121-Tenant2, mcast: 239.0.0.121 }
  - { vlan_id: 122, vni: 50122, ip_add: 10.1.122.1/24, vlan_name: L2-VNI-122-Tenant2, mcast: 239.0.0.122 }
  - { vlan_id: 123, vni: 50123, ip_add: 10.1.123.1/24, vlan_name: L2-VNI-123-Tenant2, mcast: 239.0.0.123 }

L3VNI_Tenant1:
  - { vlan_id: 999, vlan_name: L3-VNI-999-Tenant1, vni: 50999 }
L3VNI_Tenant2:
  - { vlan_id: 1000, vlan_name: L3-VNI-1000-Tenant2, vni: 51000 }
```
**group_vars/spines.yml** file:
```yaml
loopback0: 10.1.1.102/32
loopback1: 10.1.1.200/32
router_id: 10.1.1.102
eth1: 10.1.10.6/30
eth2: 10.1.10.14/30
eth3: 10.1.10.22/30
eth4: 10.1.10.30/30

L3_interfaces:
- { interface: Ethernet1/1, descr: Underlay_dc1-leaf-001_Ethernet1/2 }
- { interface: Ethernet1/2, descr: Underlay_dc1-leaf-002_Ethernet1/2 }
- { interface: Ethernet1/3, descr: Underlay_dc1-leaf-003_Ethernet1/2 }
- { interface: Ethernet1/4, descr: Underlay_dc1-leaf-004_Ethernet1/2 }

L3_loopbacks:
- { interface: loopback0, descr: RID and Update Source }
- { interface: loopback1, descr: Anycast RP }

Underlay_interfaces:
  - { underlay_int: Ethernet1/1, underlay_addr: 10.1.10.6/30, underlay_descr: Underlay_dc1-leaf-001_Ethernet1/2 }
  - { underlay_int: Ethernet1/2, underlay_addr: 10.1.10.14/30, underlay_descr: Underlay_dc1-leaf-002_Ethernet1/2 }
  - { underlay_int: Ethernet1/3, underlay_addr: 10.1.10.22/30, underlay_descr: Underlay_dc1-leaf-003_Ethernet1/2 }
  - { underlay_int: Ethernet1/4, underlay_addr: 10.1.10.30/30, underlay_descr: Underlay_dc1-leaf-004_Ethernet1/2 }

loopback1_rp: 10.1.1.200
s1_loopback: 10.1.1.101
s2_loopback: 10.1.1.102
```


## Playbooks

### (1) Global Configuration Playbook

In the global configuration playbook we define base configuration tasks that are common to all switches:
1.  Create a snapshop of the current Configuration
2.  Configure device hostnames as defined in the hosts file
3.  Enable the BGP, OSPF, Interface-VLAN, EVPN and VXLAN Features
4.  Create loopback0 interface
5.  Enable the OSPF process


Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 1_global_configuration.yml
```

```yaml
---
{% raw %}
- name: Network Getting Started First Playbook Extended
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: all
  tasks:

    - name: Create snapshot
      nxos_snapshot:
        action: create
        snapshot_name: Ansible_snapshot
        description: Done with Ansible
        save_snapshot_locally: true

    - name: remove configuration
      nxos_system:
        state: absent

    - name: configure hostname and domain-name
      nxos_system:
        hostname: "{{ hostname }}"
        domain_name: test.example.com


    - name: configure name servers
      nxos_system:
        name_servers:
          - 8.8.8.8
          - 8.8.4.4

    - name: configure name servers with VRF support
      nxos_system:
        name_servers:
          - { server: 8.8.8.8, vrf: management }
          - { server: 8.8.4.4, vrf: management }


    - name: Ensure BGP is enabled
      nxos_feature:
        feature: bgp
        state: enabled

    - name: Ensure ospf is enabled
      nxos_feature:
        feature: ospf
        state: enabled

    - name: Ensure Interface VLAN is enabled
      nxos_feature:
        feature: interface-vlan
        state: enabled

    - name: Ensure PIM is enabled
      nxos_feature:
        feature: pim
        state: enabled

    - name: Ensure VN Segment is enabled
      nxos_feature:
        feature: vn-segment-vlan-based
        state: enabled

    - name: Ensure NV Overlay is enabled
      nxos_feature:
        feature: nv overlay
        state: enabled

    - name: Enable EVPN Control Plane
      nxos_evpn_global:
        nv_overlay_evpn: true


    - name: Merge provided configuration with device configuration.
      nxos_l3_interfaces:
        config:
        - name: loopback0
          ipv4:
          - address: "{{ loopback0 }}"
        state: merged


    - name: Create OSPF instance
      nxos_ospf:
        ospf: 1
        state: present

    - name: Enable OSPF on interface
      cisco.nxos.nxos_ospf_interfaces:
        config:
          - name: loopback0
            address_family:
            - afi: ipv4
              processes:
              - process_id: "1"
                area:
                  area_id: 0.0.0.0
                  secondaries: False
{% endraw %}                  
```

### (2) Spines Playbook

The Configure Spines Playbook:
1.  Creates the global BGP routing process and iBGP neighbours for VTEP reachability
2.  Creates the loopback1 interface for multicast routing, PIM interfaces and RP-addresses
3.  COnfigures the underlay interfaces and IP addresses
4.  Enables OSPF on these interfaces

Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 2_configure_spines.yml
```

```yaml
---
  {% raw %}
- name: Configuring Spine Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: spines
  tasks:

  - name: Configure Leaf iBGP neighbours
    cisco.nxos.nxos_bgp_global:
      config:
        as_number: "{{ asn }}"
        router_id: "{{ router_id }}"
        log_neighbor_changes: True
        neighbors:
          - neighbor_address: "{{ item.neighbor }}"
            remote_as: "{{ item.remote_as }}"
            description: Leaf iBGP Neighbours
            update_source: "{{ item.update_source }}"
    with_items: "{{ bgp_neighbors }}"


  - name: Configure iBGP neighbor AF
    cisco.nxos.nxos_bgp_neighbor_address_family:
      config:
        as_number: "{{ asn }}"
        neighbors:
          - neighbor_address: "{{ item.neighbor }}"
            address_family:
              - afi: ipv4
                safi: unicast
                route_reflector_client: yes
                send_community:
                  both: yes
    with_items: "{{ bgp_neighbors }}"

  - name: Create Loopback1 Interface
    nxos_l3_interfaces:
      config:
      - name: loopback1
        ipv4:
        - address: "{{ loopback1 }}"
    with_items: "{{L3_interfaces}}"
    tags: Underlay


  - name: Create Layer 3 Interfaces
    nxos_interfaces:
      config:
      - name: "{{ item.interface }}"
        description: "{{ item.descr }}"
        mode: layer3
        enabled: true
      state: merged
    with_items: "{{L3_interfaces}}"
    tags: multicast


  - name: Set interface IPv4 address on underlay
    nxos_l3_interfaces:
      config:
      - name: "{{ item.underlay_int }}"
        ipv4:
        - address: "{{ item.underlay_addr }}"
    with_items: "{{Underlay_interfaces}}"
    tags: Underlay



  - name: Create Anycast RP interface
    cisco.nxos.nxos_interfaces:
      config:
      - name: loopback1
        description: Anycast RP interface
        enabled: true
      state: merged

  - name: Configure PIM Loopbacks
    nxos_pim_interface:
      interface: "{{ item.interface }}"
      sparse: true
    with_items: "{{L3_loopbacks}}"
    tags: multicast

  - name: Configure PIM int
    nxos_pim_interface:
      interface: "{{ item.interface }}"
      sparse: true
    with_items: "{{L3_interfaces}}"
    tags: multicast

  - name: Enable OSPF on interface
    cisco.nxos.nxos_ospf_interfaces:
      config:
        - name: "{{ item.interface }}"
          address_family:
          - afi: ipv4
            processes:
            - process_id: "1"
              area:
                area_id: 0.0.0.0
                secondaries: False
    with_items: "{{L3_interfaces}}"
    tags: ospf

  - name: Enable OSPF on Lo1
    cisco.nxos.nxos_ospf_interfaces:
      config:
        - name: "{{ item.interface }}"
          address_family:
          - afi: ipv4
            processes:
            - process_id: "1"
              area:
                area_id: 0.0.0.0
                secondaries: False
    with_items: "{{L3_loopbacks}}"
    tags: ospf

  - name: Configure PIM RP
    nxos_pim_rp_address:
      rp_address: "{{ loopback1_rp }}"
    tags: multicast

  - name: Configure Anycast RP
    nxos_config:
      lines:
       - "ip pim anycast-rp {{ loopback1_rp }} {{ s1_loopback }}"
       - "ip pim anycast-rp {{ loopback1_rp}} {{ s2_loopback }}"
    tags: multicast
    {% endraw %}
```


### (3) Leafs Playbook

The Configure Leafs Playbook:
1.  Creates the global BGP routing process and iBGP neighbours for VTEP reachability
2.  Creates the loopback1 interface for multicast routing, PIM interfaces and RP-addresses
3.  Configures the underlay interfaces and IP addresses
4.  Enables OSPF on these interfaces
5.  Configures the Access Interface E1/5 which connects to the Ubuntu hosts used in our lab


Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 3_configure_leafs.yml
```

```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: leafs
  tasks:

  - name: Merge provided configuration with device configuration.
    nxos_l3_interfaces:
      config:
      - name: loopback1
        ipv4:
        - address: "{{ loopback1 }}"
    with_items: "{{L3_interfaces}}"
    tags: Underlay

  - name: Create Layer 3 Interfaces
    nxos_interfaces:
      config:
      - name: "{{ item.interface }}"
        description: "{{ item.descr }}"
        mode: layer3
        enabled: true
      state: merged
    with_items: "{{L3_interfaces}}"
    tags: multicast

  - name: Enable OSPF on interface
    cisco.nxos.nxos_ospf_interfaces:
      config:
        - name: "{{ item.interface }}"
          address_family:
          - afi: ipv4
            processes:
            - process_id: "1"
              area:
                area_id: 0.0.0.0
                secondaries: False
    with_items: "{{L3_interfaces}}"
    tags: ospf

  - name: Enable OSPF on Lo1
    cisco.nxos.nxos_ospf_interfaces:
      config:
        - name: "{{ item.interface }}"
          address_family:
          - afi: ipv4
            processes:
            - process_id: "1"
              area:
                area_id: 0.0.0.0
                secondaries: False
    with_items: "{{L3_loopbacks}}"
    tags: ospf

  - name: Set interface IPv4 address on underlay
    nxos_l3_interfaces:
      config:
      - name: "{{ item.underlay_int }}"
        ipv4:
        - address: "{{ item.underlay_addr }}"
    with_items: "{{Underlay_interfaces}}"
    tags: Underlay


  - name: Configure Access Interfaces
    cisco.nxos.nxos_l2_interfaces:
      config:
      - name: Ethernet1/5
        mode: access
        access:
          vlan: "{{access_vlan}}"
      state: merged

  - name: Enable PIM
    nxos_feature:
      feature: pim
      state: enabled
    tags: multicast

  - name: Create Anycast RP interface
    cisco.nxos.nxos_interfaces:
      config:
      - name: loopback1
        description: Anycast RP interface
        enabled: true
      state: merged

  - name: Merge provided configuration with device configuration.
    nxos_l3_interfaces:
      config:
      - name: loopback1
        ipv4:
        - address: "{{ loopback1 }}"
    with_items: "{{L3_interfaces}}"
    tags: Underlay


  - name: Configure PIM int
    nxos_pim_interface:
      interface: "{{ item.interface }}"
      sparse: true
    with_items: "{{L3_interfaces}}"
    tags: multicast

  - name: Configure PIM Loopbacks
    nxos_pim_interface:
      interface: "{{ item.interface }}"
      sparse: true
    with_items: "{{L3_loopbacks}}"
    tags: multicast

  - name: Configure PIM RP
    nxos_pim_rp_address:
      rp_address: "{{ rp_address }}"
    tags: multicast

  - name: Configure Leaf iBGP neighbours
    cisco.nxos.nxos_bgp_global:
      config:
        as_number: "{{ asn }}"
        router_id: "{{ router_id }}"
        log_neighbor_changes: True
        neighbors:
          - neighbor_address: "{{ item.neighbor }}"
            remote_as: "{{ item.remote_as }}"
            description: Leaf iBGP Neighbours
            update_source: "{{ item.update_source }}"
    with_items: "{{ bgp_neighbors }}"


  - name: Configure iBGP neighbor AF
    cisco.nxos.nxos_bgp_neighbor_address_family:
      config:
        as_number: "{{ asn }}"
        neighbors:
          - neighbor_address: "{{ item.neighbor }}"
            address_family:
              - afi: ipv4
                safi: unicast
                route_reflector_client: yes
                send_community:
                  both: yes
    with_items: "{{ bgp_neighbors }}"
    {% endraw %}
```

### (4) BGP EVPN Spines Playbook

BGP EVPN Spines Playbook:
1.  Creates the EVPN BGP Address Family
2.  Configure Route Reflector Clients

Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 4_configure_bgp_evpn_spines.yml
```

```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: spines
  tasks:
    - name: Configure BGP EVPN
      nxos_bgp_af:
       asn: "{{ asn }}"
       afi: l2vpn
       safi: evpn
      tags: evpn
    - name: Configure iBGP neighbor EVPN AF
      nxos_bgp_neighbor_af:
       asn: "{{ asn }}"
       neighbor: "{{ item.neighbor }}"
       afi: l2vpn
       safi: evpn
       route_reflector_client: "true"
       send_community: both
      with_items: "{{ bgp_neighbors }}"
      tags: evpn
      {% endraw %}
```

### (5) BGP EVPN Leafs Playbook

BGP EVPN Leafs Playbook:
1.  Creates the EVPN BGP Address Family
2.  Configure Spines as Neighbours
3.  Iterate over the list of VLANs in the **leafs.yml** file in group_vars and define the EVPN VNI


Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 5_configure_bgp_evpn_leafs.yml
```

```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: leafs
  tasks:
    - name: Configure BGP EVPN
      nxos_bgp_af:
       asn: "{{ asn }}"
       afi: l2vpn
       safi: evpn
      tags: evpn
    - name: Configure iBGP neighbor EVPN AF
      nxos_bgp_neighbor_af:
       asn: "{{ asn }}"
       neighbor: "{{ item.neighbor }}"
       afi: l2vpn
       safi: evpn
       send_community: both
      with_items: "{{ bgp_neighbors }}"
      tags: evpn
    - name: Configure L2VNI_Tenant2 RD/RT
      nxos_evpn_vni:
       vni: "{{ item.vni }}"
       route_distinguisher: auto
       route_target_both: auto
      with_items: "{{ L2VNI_Tenant2 }}"
      tags: evpn
    - name: Configure L2VNI_Tenant1 RD/RT
      nxos_evpn_vni:
       vni: "{{ item.vni }}"
       route_distinguisher: auto
       route_target_both: auto
      with_items: "{{ L2VNI_Tenant1 }}"
      tags: evpn
      {% endraw %}
```

### (6) Tenant-1 VXLAN Playbook

In Playbooks 6 and 7, we define the VLAN to VNI mapping, interface VLANs and IP Addressing and create the NVE interface or VTEP tunnel on each leaf used for our two example Tenants "Tenant-1" and "Tenant-2".

Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 6_configure_vxlan_leafs_tenant_1.yml
```

```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: leafs
  tasks:
   - name: Enable VXLAN Feature
     nxos_feature:
       feature: "{{ item }}"
       state: enabled
     with_items:
      - nv overlay
      - vn-segment-vlan-based
     tags: vxlan
   - name: Enable NV Overlay
     nxos_evpn_global:
       nv_overlay_evpn: true
     tags: vxlan
   - name: Configure VLAN to VNI
     nxos_vlan:
       vlan_id: "{{ item.vlan_id }}"
       mapped_vni: "{{ item.vni }}"
       name: "{{ item.vlan_name }}"
     with_items:
     - "{{ L2VNI_Tenant1 }}"
     - "{{ L3VNI_Tenant1 }}"
     tags: vxlan
   - name: Configure Tenant VRF
     nxos_vrf:
       vrf: Tenant-1
       rd:  auto
       vni: "{{ L3VNI_Tenant1[0].vni }}"
     tags: vxlan
   - name: Configure VRF AF
     nxos_vrf_af:
       vrf: Tenant-1
       route_target_both_auto_evpn: true
       afi: ipv4
     tags: vxlan
   - name: Configure Anycast GW
     nxos_overlay_global:
       anycast_gateway_mac: 0000.2222.3333
     tags: vxlan
   - name: Configure L2VNI_Tenant1
     nxos_interface:
       interface: vlan"{{ item.vlan_id }}"
     with_items: "{{ L2VNI_Tenant1 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant1
     nxos_interface:
       interface: vlan"{{ L3VNI_Tenant1[0].vlan_id }}"
     tags: vxlan
   - name: Assign interface to Tenant VRF
     nxos_vrf_interface:
       vrf: Tenant-1
       interface: "vlan{{ item.vlan_id }}"
     with_items:
     - "{{ L2VNI_Tenant1 }}"
     - "{{ L3VNI_Tenant1 }}"
     tags: vxlan



   - name: Set interface IPv4 address on underlay
     nxos_l3_interface:
       name: "vlan{{ item.vlan_id }}"
       ipv4: "{{ item.ip_add }}"
     with_items: "{{L2VNI_Tenant1}}"
     tags: vxlan
   - name: Configure L2VNI_Tenant1 SVI
     nxos_interface:
       interface: vlan"{{ item.vlan_id }}"
       fabric_forwarding_anycast_gateway: true
     with_items: "{{ L2VNI_Tenant1 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant1 SVI
     nxos_interface:
       interface: vlan"{{ L3VNI_Tenant1[0].vlan_id }}"
       ip_forward: enable
     tags: vxlan
   - name: Configure VTEP Tunnel
     nxos_vxlan_vtep:
        interface: nve1
        shutdown: "false"
        source_interface: Loopback1
        host_reachability: "true"
     tags: vxlan
   - name: Configure L2VNI_Tenant1 to VTEP
     nxos_vxlan_vtep_vni:
        interface: nve1
        vni: "{{ item.vni }}"
        multicast_group: "{{ item.mcast }}"
     with_items: "{{ L2VNI_Tenant1 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant1 to VTEP
     nxos_vxlan_vtep_vni:
        interface: nve1
        vni: "{{ L3VNI_Tenant1[0].vni }}"
        assoc_vrf: true
     tags: vxlan
{% endraw %}
```

### (7) Tenant-2 VXLAN Playbook


Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 7_configure_vxlan_leafs_tenant_2.yml
```
```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: leafs
  tasks:
   - name: Configure VLAN to VNI
     nxos_vlan:
       vlan_id: "{{ item.vlan_id }}"
       mapped_vni: "{{ item.vni }}"
       name: "{{ item.vlan_name }}"
     with_items:
     - "{{ L2VNI_Tenant2 }}"
     - "{{ L3VNI_Tenant2 }}"
     tags: vxlan
   - name: Configure Tenant VRF
     nxos_vrf:
       vrf: Tenant-1
       rd:  auto
       vni: "{{ L3VNI_Tenant2[0].vni }}"
     tags: vxlan
   - name: Configure VRF AF
     nxos_vrf_af:
       vrf: Tenant-2
       route_target_both_auto_evpn: true
       afi: ipv4
     tags: vxlan
   - name: Configure Anycast GW
     nxos_overlay_global:
       anycast_gateway_mac: 0000.2222.3333
     tags: vxlan
   - name: Configure L2VNI_Tenant2
     nxos_interface:
       interface: vlan"{{ item.vlan_id }}"
     with_items: "{{ L2VNI_Tenant2 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant2
     nxos_interface:
       interface: vlan"{{ L3VNI_Tenant2[0].vlan_id }}"
     tags: vxlan
   - name: Assign interface to Tenant VRF
     nxos_vrf_interface:
       vrf: Tenant-2
       interface: "vlan{{ item.vlan_id }}"
     with_items:
     - "{{ L2VNI_Tenant2 }}"
     - "{{ L3VNI_Tenant2 }}"
     tags: vxlan
   - name: Set interface IPv4 address on underlay
     nxos_l3_interface:
       name: "vlan{{ item.vlan_id }}"
       ipv4: "{{ item.ip_add }}"
     with_items: "{{L2VNI_Tenant2}}"
     tags: vxlan
   - name: Configure L2VNI_Tenant2 SVI
     nxos_interface:
       interface: vlan"{{ item.vlan_id }}"
       fabric_forwarding_anycast_gateway: true
     with_items: "{{ L2VNI_Tenant2 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant2 SVI
     nxos_interface:
       interface: vlan"{{ L3VNI_Tenant2[0].vlan_id }}"
       ip_forward: enable
     tags: vxlan
   - name: Configure VTEP Tunnel
     nxos_vxlan_vtep:
        interface: nve1
        shutdown: "false"
        source_interface: Loopback1
        host_reachability: "true"
     tags: vxlan
   - name: Configure L2VNI_Tenant2 to VTEP
     nxos_vxlan_vtep_vni:
        interface: nve1
        vni: "{{ item.vni }}"
        multicast_group: "{{ item.mcast }}"
     with_items: "{{ L2VNI_Tenant2 }}"
     tags: vxlan
   - name: Configure L3VNI_Tenant2 to VTEP
     nxos_vxlan_vtep_vni:
        interface: nve1
        vni: "{{ L3VNI_Tenant2[0].vni }}"
        assoc_vrf: true
     tags: vxlan
{% endraw %}
```

### (8) Fabric Verification Playbook

Use the "cisco.nxos.nxos_command" task from the NXOS collection to run verification commands against our fabric.

On each switch we use the Ansible **debug** module to return the output of these commands to the terminal.

1.  NXOS version
2.  Interface status
3.  Enabled Features
4.  CDP neighbours
5.  VLAN Status
6.  MAC address-table to verify our Ubuntu host is connected to the leaf switches and the access port is up.
7.  iBGP Neighbours
8.  EVPN Neighbours
9.  VTEP/NVE Interface status
10.  MAC and IP learning and host routes in the EVPN address family
11.  Tenant-1 routes
12.  Tenant-2 Routes
13.  Connectivity tests for underlay addresses and anycast SVIs for Tenant-1 and Tenant-2


Running the playbook:
```shell
$ ansible-playbook -i hosts -c ansible.netcommon.network_cli 8_verify_fabric.yml
```

```yaml
---
{% raw %}
- name: Configuring Leaf Switches
  connection: ansible.netcommon.network_cli
  gather_facts: false
  hosts: leafs
  tasks:
    - name: (1) Check Running Version
      register: version_output
      cisco.nxos.nxos_command:
        commands:
        - command: show version
        wait_for:
          - result[0] contains bootflash:///nxos.9.3.7.bin
    - debug: var=version_output.stdout_lines


    - name: (2) Interface Status
      register: interface_output
      cisco.nxos.nxos_command:
        commands:
        - command: sh ip int br | exclude unass
        wait_for:
          - result[0] contains Lo0
    - debug: var=interface_output.stdout_lines

    - name: (3) Enabled Features
      register: feature_output
      cisco.nxos.nxos_command:
        commands:
        - command: show feature | grep enabled
        wait_for:
          - result[0] contains interface-vlan
    - debug: var=feature_output.stdout_lines


    - name: (4) CDP Neighours
      register: cdp_output
      cisco.nxos.nxos_command:
        commands:
        - command: show cdp neighbors | excl mgmt0
        wait_for:
          - result[0] contains Eth1/1
    - debug: var=cdp_output.stdout_lines


    - name: (5) VLAN Status
      register: vlan_output
      cisco.nxos.nxos_command:
        commands:
        - command: show vlan
        wait_for:
          - result[0] contains active
    - debug: var=vlan_output.stdout_lines


    - name: (6) MAC Address Table
      register: mac_output
      cisco.nxos.nxos_command:
        commands:
        - command: show mac address-table | exclude sup
        wait_for:
          - result[0] contains MAC Address
    - debug: var=vlan_output.stdout_lines


    - name: (7) iBGP Neighbours
      register: ibgp_output
      cisco.nxos.nxos_command:
        commands:
        - command: show ip bgp summary
        wait_for:
          - result[0] contains AS number 65000
    - debug: var=ibgp_output.stdout_lines


    - name: (8) EVPN Neighbours
      register: evpn_output
      cisco.nxos.nxos_command:
        commands:
        - command: show bgp l2vpn evpn summary
        wait_for:
          - result[0] contains AS number 65000
    - debug: var=evpn_output.stdout_lines

    - name: (9) NVE Interface
      register: nve_output
      cisco.nxos.nxos_command:
        commands:
        - command: show nve interface
        wait_for:
          - result[0] contains VXLAN
    - debug: var=nve_output.stdout_lines

    - name: (10) Check evpn MAC and IP learning
      register: l2route_output
      cisco.nxos.nxos_command:
        commands:
        - command: show l2route evpn mac-ip all
        wait_for:
          - result[0] contains Orphan
    - debug: var=l2route_output.stdout_lines

    - name: (11) Check Tenant-1 Routes
      register: Tenant1_output
      cisco.nxos.nxos_command:
        commands:
        - command: show ip route vrf Tenant-1
        wait_for:
          - result[0] contains IP Route Table
    - debug: var=Tenant1_output.stdout_lines

    - name: (12) Check Tenant-2 Routes
      register: Tenant2_output
      cisco.nxos.nxos_command:
        commands:
        - command: show ip route vrf Tenant-2
        wait_for:
          - result[0] contains IP Route Table
    - debug: var=Tenant2_output.stdout_lines

    - name: (11) Connectivity
      register: ping_output
      cisco.nxos.nxos_command:
        commands:
        - command: ping 10.1.1.101
        - command: ping 10.1.1.102
        - command: ping 10.1.1.200
        - command: ping 10.1.10.10 vrf Tenant-1
        - command: ping 10.1.10.11 vrf Tenant-1
        - command: ping 10.1.110.1 vrf Tenant-2
        - command: ping 10.1.111.1 vrf Tenant-2
        wait_for:
          - result[0] contains icmp_seq
    - debug: var=ping_output.stdout_lines
{% endraw %}
```
