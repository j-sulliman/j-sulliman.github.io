---
layout: post
author: Jamie Sullivan
date:  2021-07-20 08:00:00 +1200
---

# Introduction

In a [previous article](https://j-sulliman.github.io/2021/05/04/Part-3.1-DCNM-Lab-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) we stepped through creating a virtualised EVPN and VXLAN lab using Cisco's controller based DCNM and N9000vs to automate deployment, configuration templating and consolidate management and monitoring.

In this article, use Ansible to automate the configuration and provisioning of an EVPN and VXLAN fabric.








# Resources
* The free introductory Ansible course available from [RedHat](https://www.redhat.com/en/blog/new-free-ansible-course).
* The structured labs and learning available for Ansible that can be found on [DevNet](https://developer.cisco.com/automation-ansible/).  
* Create your own automation labs using the collections published on [docs.ansible.com](https://docs.ansible.com/ansible/latest/collections/cisco/index.html).

# Ansible Overview and Setup

An overview of some of the ansible terminology I'll refer to through this article:

* **Tasks**: A command executed against a host.
* **Playbooks**: A sequence of commands executed against a host. We'll use eight playbooks for our EVPN and VXLAN deployment.
* **Variables**:  Configuration parameters that are unique to each host (e.g. Loopback0 address) are defined in variables, which can be defined in multiple places.  The playbooks we'll use reference variables in the hosts (inventory file), host_vars and group_vars directories - see the directory structure below for an example.  You can read more on variables in the [User Guide](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#ansible-variable-precedence)
* **Hosts File**:  An inventory of devices that playbooks will be run against is defined in the **hosts** file.  Our hosts file will include two groups [spines] and [leafs].  Within the hosts file you can also define aliases and host vars.  Loopback addresses and hostnames for example.  Defining host specific variables in the hosts file soon becomes unwieldily to manage as the number of variables grow.  We'll look at some alternatives - group_vars and host_vars below.
* **Collections**: [Cisco Nxos collection](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/index.html#plugins-in-cisco-nxos)

# Lab Overview


Below shows the fabric we'll deploy using Ansible Playbooks.

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

# Directory Structure Overview
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
< >

## Inventory and Hosts file
<>

## Variables
<>

### Inventory vars
<>

### Host vars

<>

### Group vars

<>

## Playbooks


# 7.2 References

* [ACI on DevNet](https://developer.cisco.com/site/aci/)
* [ACI GitHub Repos](https://github.com/search?q=cisco+aci)
* [APIC REST API Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/2-x/rest_cfg/2_1_x/b_Cisco_APIC_REST_API_Configuration_Guide/b_Cisco_APIC_REST_API_Configuration_Guide_chapter_01.html)


# Demo
<iframe width="560" height="315" src="https://www.youtube.com/embed/{{ include.id }}" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>




##### Why Ansible?

Infrastructure as code platforms such as Ansible, are relatively easy for non-programmers to maintain.  For operations and devops environments where templating and automation needs to be managed by a wider team of varying skillsets on an ongoing basis, the automation tools need to be easy to be easy to maintain, re-usable by others and well documented.

For project based implementation, deployment and migration scenarios - where configuration dues not need to be managed on an ongoing basis. I.e. Deliverables  with clearly defined implementation milestones the approach to automation is driven more by personal preference and familiarity.
