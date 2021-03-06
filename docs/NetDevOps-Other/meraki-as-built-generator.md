---
layout: default
title: Meraki As-Built Generator
parent: NetDevOps -  all other articles
nav_order: last
last_modified_date: 2021-07-19 08:00:00 +1200
---

[![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/j-sulliman/mabg)

# meraki

# MABG - Meraki AS Built Generator: Automation of AS Built Document for Meraki

The application should reduce the time needed by the network engineer to manually document the meraki network when handing over to customer/operations/VAR consulting teams after implementation.

Retrieves Meraki configuration via an API call, stores returned data and writes to a word document for further
editing and customisation.


# Demo

<iframe width="700" height="400" src="https://www.youtube.com/embed/0SyVrn82g6g" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>



# Setup

1) Ensure Meraki API Access is enabled
![alt text](https://github.com/j-sulliman/meraki/blob/master/AsBuilts/media/Enable-API.png?raw=true)


# Create virtual environment / sandbox
```
JASULLI2-M-F04H:Programming jasulli2$ cd meraki_asbuilt/
JASULLI2-M-F04H:meraki_asbuilt jasulli2$ ls
JASULLI2-M-F04H:meraki_asbuilt jasulli2$ python3 -m venv venv
JASULLI2-M-F04H:meraki_asbuilt jasulli2$ source venv/bin/activate
```

# Initialise git and pull remote MABG repository
```
(venv) JASULLI2-M-F04H:meraki_asbuilt jasulli2$ git init
Initialized empty Git repository in /Users/jasulli2/OneDrive - Cisco/Programming/meraki_asbuilt/.git/
(venv) JASULLI2-M-F04H:meraki_asbuilt jasulli2$ git pull https://github.com/j-sulliman/meraki.git

remote: Enumerating objects: 1511, done.
remote: Counting objects: 100% (1511/1511), done.
remote: Compressing objects: 100% (1153/1153), done.
remote: Total 1511 (delta 329), reused 1463 (delta 284), pack-reused 0
Receiving objects: 100% (1511/1511), 5.95 MiB | 2.13 MiB/s, done.
Resolving deltas: 100% (329/329), done.
From https://github.com/j-sulliman/meraki
 * branch            HEAD       -> FETCH_HEAD
```

# Install the python dependencies
```
(venv) JASULLI2-M-F04H:meraki_asbuilt jasulli2$
(venv) JASULLI2-M-F04H:meraki_asbuilt jasulli2$ pip3 install -r requirements.txt


Successfully installed Django-2.2.4 bootstrap4-0.1.0 certifi-2019.6.16 chardet-3.0.4 django-tables2-2.1.0 et-xmlfile-1.0.1 idna-2.8 jdcal-1.4.1 lxml-4.4.0 numpy-1.17.0 openpyxl-2.6.2 pandas-0.25.0 pprint-0.1 python-dateutil-2.8.0 python-docx-0.8.10 pytz-2019.1 requests-2.22.0 six-1.12.0 sqlparse-0.3.0 urllib3-1.25.3
```

# Start Django Server
```
$ cd AsBuilts/
$ python3 manage.py runserver 0:8080

Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
Starting development server at http://0:8080/
Quit the server with CONTROL-C.
```

# Browse to main page and enter API Key and Customer Name
i.e. http://127.0.0.1:8080/meraki/defaults/
![alt text](https://github.com/j-sulliman/meraki/blob/master/AsBuilts/media/main-page.png?raw=true)

Download and review the generated As Built Document
![alt text](https://github.com/j-sulliman/meraki/blob/master/AsBuilts/media/Document-Example.png?raw=true)


Optionally review and delete the Logfile / API call status codes appended to the end of document
![alt text](https://github.com/j-sulliman/meraki/blob/master/AsBuilts/media/Log-File.png?raw=true)


# Disclaimer
Please review and edit retrieved data as required.  Feedback welcomed.
