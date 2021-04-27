---
layout: post
author: Jamie Sullivan
date:  2021-04-27 08:00:34 +1200
---
# Part 1:  Introduction - Network and Infra Monitoring with Grafana, Django and Python
Cisco continues to work towards increasing cross domain integrations (DNAC and Meraki, DNAC and vManage, Nexus Dashboard), but there isn't **yet** an *Uber Multi-Domain Controller* that I know of.  

What Cisco does have, is best in class APIs across architectures and a great set of resources in DevNet that allow Enterprises, MSPs, DevOps teams, or those that just like to automate, a high degree of flexibility when automating their environments.

In this multi-part blog series, we use REST APIs, Python, Django Webframework, PostgresSQL and Grafana to deliver a multi-domain API based monitoring and visualisations Dashboard for a range of network and Infrastructure domains.

##### Fault Aggregation:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Grafana.Dashboard.png?raw=true)



#### Part 2:  Overview
* Resources
* Django Webframework
* Grafana
* Test Environments

#### Part 3: Campus LAN Use Case
* (a) Cisco Meraki
##### Meraki Devices:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki.Devices.png?raw=true)


* (b) Cisco DNA-C
##### DNAC Devices:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Screen%20Shot%202021-04-27%20at%209.23.26%20AM.png?raw=true)

#### Part 4: Software Defined WAN Use Case
* Cisco SDWAN (Viptella)
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/SDWAN.png?raw=true)

#### Part 5: Datacenter Network Use Case
* (a) Cisco ACI
* (b) DCNM VXLAN and EVPN Fabric

#### Part 6: Datacenter Compute Use Case
* Cisco UCS

#### Part 7: Vulnerability Monitoring Use Case
* Openvuln API
#### Openvuln API Dashboard
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/openvuln.png?raw=true)
