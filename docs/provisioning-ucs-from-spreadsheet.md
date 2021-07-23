---
layout: default
title: Provisioning UCS from Spreadsheet
parent:
nav_order: last
---

[![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/j-sulliman/pufs)

# PUFS - Provision UCS from Spreadsheet
Get a headstart on UCS design and deployment
Reduce implementation time from days to minutes
Reduce configuration errors by testing deployment against pre-prod in minutes
Auto generate design documentation

# Suggested Workflow
Gather requirements and design inputs for UCS implementation
Enter into spreadsheet (ucs-implementation-data.xlsx) - this spreadsheet would be the source of truth for UCS design and implementation detail
Pre production implementation - Push the configuration parameters / inputs to UCS Platform Emulator / non production environment and validate and adjust as needed

![alt text](https://github.com/j-sulliman/pufs/blob/master/Spreadsheet.png)

A Design Document in word format will be automatically generated - this can be used as a starting point for Detailed Design or As Builts
Once satisified with pre-prod (or UCS Platform Emulator) configuration push to production


![alt text](https://github.com/j-sulliman/pufs/blob/master/UCS%20Design%20Document.png)

# Demo
<iframe width="700" height="400" src="https://www.youtube.com/embed/nKs3KvOY_Tw" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>


# Setup
```
$ python3 -m venv venv

$ source venv/bin/activate

$ git init

$ git pull https://github.com/j-sulliman/pufs.git

Install the required dependencies:

$ pip3 install -r requirements.txt
```

# Running the Script
Run ucs_app.py with UCSM IP, Username, Password and input spreadsheet arguements as below
```
$ python ucs_app.py -a <192.168.100.132> -u <ucspe> -p <ucspe> -f <ucs-implementation-data.xlsx>
or
python ucs_app.py -a <UCSM IP> -u <UCSM Username> -p <UCSM Password> -f <spreadsheet name.xlsx>

```

# Validate the configuration in UCSM and Review the generated word document
![alt text](https://github.com/j-sulliman/pufs/blob/master/UCSM.png)

# Assumptions
Script has been tested against the following UCSM software versions:
(The UCS Platform Emulator can be downloaded from here: https://community.cisco.com/t5/unified-computing-system/ucs-platform-emulator-downloads-ucspe-4-1-2cpe1-ucspe-4-0-4epe1/ta-p/3648177)
```
UCS Platform Emulator: UCSPE_4.0.4e.ovf, UCSPE_3.2.3e.ovf
UCSM 3.2.3, 4.0.x
```
This repository contains a working sample spreadsheet with sheets containing configuration parameters for a range of UCSM Policies.  These can be edited to suit your implementation.  Note that any incorrect references/names/values in the spreadsheet will also result in a misconfigured UCSM policy or error when running the script.  You may want to use cell references and validation to capture any invalid input
