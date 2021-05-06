---
layout: post
author: Jamie Sullivan
date:  2021-05-05 08:00:34 +1200
---
# Part (3a) DCNM and VXLAN BGP and EVPN Lab with Nexus 9000v
In this update we outline of the lab topology used as the automation sandbox to explore APIs and Visualisations with Grafana and DCNM.  If you already have a DCNM environment available, you can skip this page and continue to Part 3b.


| Section | Architecture | Link | Topic
------------ | ------------ | ------------- | -------------
1 | Introduction | [building a multidomain dashboard with cisco apis and grafana](https://j-sulliman.github.io/2021/04/26/Part-1-Intro-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Solution Overview
2 | Meraki | [Building a Meraki Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/04/26/Part-2-Meraki-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising Meraki APIs with Grafana.
3a | DCNM & EVPN VXLAN | [DCNM and VXLAN BGP and EVPN Lab with Nexus 9000v](https://j-sulliman.github.io/2021/04/26/Part-3a-DCNM-Lab-Build-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | DCNM, N9000v, EVPN and VXLAN Lab deployment
3b | DCNM & EVPN VXLAN | [Building a Meraki Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/04/26/Part-2-Meraki-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising DCNM APIs with Grafana.


##### DCNM VXLAN BGP and EVPN Lab Overview
![alt text](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/DCNM_EVPN_Topology.png)


##### DCNM VXLAN BGP and EVPN Lab Components

Appliance | Image name | Version | Memory | CPU | Disk | Qty
------------ | ------------- | ------------- | ------------- | ------------- | ------------- | -------------
DCNM | dcnm-va.11.5.1.iso | 11.5 | 24768MB | 8vCPUs | 100G | 1
n9000v | nexus9300v.9.3.7.qcow2 | 9.3.7 | 8096MB | 2vCPUs | - | 10

*Remove or add nexus 9000v to suit.*



#### DCNM VXLAN BGP and EVPN Resources
* [DCNM REST API Reference Guide](https://developer.cisco.com/docs/data-center-network-manager/11-5-1/)
* [Easy Provisioning of VXLAN BGP EVPN Fabrics](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/managing-greenfield-vxlan-fabric.html)
* [Auto-Provisioning Border Gateways with Multi-Site Domains](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/border-provisioning-multisite.html )
* [Cisco Communities - Data Center](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/border-provisioning-multisite.html )




#### Pre-requisites and Guidelines
Note the following pre-requisites from the MSD configuration guide

"To ensure consistency across fabrics, ensure the following:
* The *underlay IP addresses across the fabrics*, the loopback 0 address and the loopback 1 address subnets should be *unique*.
* Ensure that each fabric has a *unique IP address pool* to avoid duplicates.
* Each fabric should have a *unique site ID and BGP AS number* associated and configured.
* All fabrics should have the *same Anycast Gateway MAC* address."

#### Base Configuration - hostname, management address and boot image
```
conf t
interface mgmt 0
  ip address [192.168.200.229/24]
!
ip route 0.0.0.0 0.0.0.0 [192.168.200.1] vrf management
hostname [dc2-bdlf-001]
boot nxos bootflash:///nxos.9.3.7.bin
copy run st
```
__Note__ – When using DHCP assigned address on Mgmt0 without POAP, I encountered *“Error during configuration read or intent”* error when Saving and Deploying configuration. The workaround was to statically assign IP addresses to Mgmt0.

### Site 1 "Auckland Fabric" Deployment and the Easy_Fabric template
From the *Control* menu, select *Fabric Builder*, *Create Fabric*, name the fabric and provide a BGP ASN
![alt text](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_1.png)

As we are using N9000v’s for our lab,  Under advanced __“Greenfield Cleanup"__ – select __“enable”__

From the configuration guide linked above -
*“Greenfield Cleanup Option – Enable the switch cleanup option for switches imported into DCNM with Preserve-Config=No, without a switch reload. This option is typically recommended only for the fabric environments with Cisco Nexus 9000v Switches to improve on the switch clean up time.*

![alt text](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_easy_frabric_2.png)

On the Resources tab – either accept defaults for the first fabric, or if connecting multiple sites, note the IP and VNI ranges as these will need to be unique to each site.

Defaults are fine for Manageability , Bootstrap, Configuration Backup

####  Add switches to Site 1 "Auckland Fabric"
Management address, username and password define in *base configuration*
![Add Switches](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_add_switches.png)

Select *manageable* switches and __Import to Fabric__
![Import Switches](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_add_switches_3.png)

Right click on spines and set roles, in my case I also have a border leaf that I added after the initial deployment, click __“Save and Deploy”__
![Assign Roles](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_assign_roles.png)

Deploy the Configuration
![Deploy Config](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_push_config.png)


### Site 2 "Wellington Fabric" Deployment and the Easy_Fabric template
Create the fabric for the second site, defining unique subnet ranges and BGP ASNs as described in the pre-requisites

![Add Switches](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_fabric_two_1.png)


Define the spine and Border Gateway Roles as required and *Deploy Config* - site two should now be __In Sync__
![Fabric Two](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_fabric_two_3.png)


### Multi-Site Domain deployment
![one](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/msd_1.png)

![two](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_msd_2.png)

![three](https://raw.githubusercontent.com/j-sulliman/j-sulliman.github.io/master/images/dcnm_msd_3.png)

![four](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_Tab_view.png?raw=true)
