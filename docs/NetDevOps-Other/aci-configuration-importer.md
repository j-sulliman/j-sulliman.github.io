---
layout: default
title: Nexus to ACI Configuration Importer
parent: NetDevOps -  all other articles
nav_order: last
last_modified_date: 2021-07-19 08:00:00 +1200
---

[![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/j-sulliman/acici)

# ACICI: Automate ACI configuration of Legacy Logical Network
Imports L2 constructs (VLANs, SVIs and associated VRF) from Legacy NXOS configuration, into ACI in a network centric approach (legacy vlan = ACI BD/EPG).

Script is VRF aware, EPGS/BDs will be grouped under Application Endpoint Groups, split by VRF.  If there are no VRFs in legacy, all EPGs/BDS will be assigned the "DEFAULT" vrf. It's up to you to create the fabric access policies.

EPGS are created with a static path binding to the legacy switch - the hostname from configuration file is used for this.  Legacy items are imported into Django models DB
Bulk clean/naming of data (e.g EPG/BD/VRFs naming convention)

Optional editing and review of imported data through Django admin front end (i.e. http 127.0.0.1:8080/admin)  

Post imported objects to APIC (EPGS are be default included in a preferred group)

![overview](https://github.com/j-sulliman/acici/blob/master/overview.png?raw=true)

# Demo
<iframe width="700" height="400" src="https://www.youtube.com/embed/V_Qyy2QHGVY" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>


# Setup
```
$ python3 -m venv venv

$ source venv/bin/activate

$ git init

$ git pull https://github.com/j-sulliman/acici.git

Install the required dependencies:

$ pip3 install -r requirements.txt
```

# Start Django Server
```
$ python3 manage.py runserver 0:8080

Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
July 18, 2019 - 13:17:42
Django version 2.2.2, using settings 'nxos_aci.settings'
Starting development server at http://0:8080/
Quit the server with CONTROL-C.
```

# Logon and Import NXOS file
i.e. http://127.0.0.1:8080
Login as admin/C1sc0123

Menu --> Upload Config File

![alt text](https://github.com/j-sulliman/acici/blob/master/acici-upload.png?raw=true)

Provide the defaults for configuration naming convention and BD construct.  BD mode in most cases should be l2 which will enable ARP and BUM flooding.  L3 mode will enable unicast routing and configure the SVI address as a BD Subnet.  EPGs will be created as "Preferred group - Include" members.

![alt text](https://github.com/j-sulliman/acici/blob/master/tenant-defaults.png?raw=true)


View and Edit the Imported configuration
![alt text](https://github.com/j-sulliman/acici/blob/master/imported-epgs.png?raw=true)

Enter the APIC connection info and submit
![alt text](https://github.com/j-sulliman/acici/blob/master/push-to-apic.png?raw=true)


View the resulting JSON and HTTP Post status code
![alt text](https://github.com/j-sulliman/acici/blob/master/epg-json.png?raw=true)
- Object configuration and DN/URL can be used with other REST API clients - i.e. postman, curl, or paste directly into APIC

Check the APIC
![alt text](https://github.com/j-sulliman/acici/blob/master/apic-import.png?raw=true)

# Create associated fabric access policies and L3Os manually
Rationale - items like Physical domain, vlan pools to legacy network will likely only be configured once.  
Fabric access policies therefore less far less time consuming than tenant policy.

L3O configuration is environment dependant.

# Disclaimer
Tested against NXOS 7.X configuration files, may work with IOS but needs testing.
Use at your own risk - recommend dry-run against a simulator or non-prod APIC
