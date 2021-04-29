---
layout: post
author: Jamie Sullivan
date:  2021-04-27 08:00:34 +1200
---
# Introduction - Network and Infra Monitoring with Grafana, Django and Python

Cisco has an API First approach across its technology domains that allows Enterprises, MSPs, DevOps practitioners the flexibility to automate their networks and infrastructure and to integrate with other platforms.

In this multi-part blog series, we use REST APIs, Python, Django Webframework, PostgresSQL and Grafana to demonstrate building a multi-domain visualisations Dashboard for a range of network and Infrastructure domains using REST APIs.

##### Solution Overview
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Overview_1.png?raw=true)

### Overview and Functional Components

#### Django
* **Extensible and Flexible**: Django allow's additional functionality to be added (e.g. user input, data presentation) as required
* **Database Abstraction**: Django's SQL abstraction API means we can work with databases in pythonic way.  Database tables are defined in static, *Model Files* and then pushed/ commited to the database... one less language (SQL) to learn.
* **Familiarity**: Access to tutorials and documentation

**Resources**
* Getting started: [Django Tutorial](https://docs.djangoproject.com/en/3.2/intro/tutorial01/)

#### Postgres SQL

* **Tool availability**:  Tools such as PgAdmin reduce complexity and knowledge needed to interact with and validate databases
* **Integration**: SQLLite is the default database deployed with Django, Postgres is also fully supported by both the Django models/database API and Grafana for visualisations.
* **Scalability**
* **Simple requirements**:  For a mock lab and self learning, Postgres SQL has more capability than I need. For production environments you'll want to analyse requirements specific to your usecase and understand the pros and cons of timeseries databases such as **InfluxDB**, **prometheus** and **Logstash** for event/syslog aggregation.  

**Resources**
* [Postgres Installation](https://www.postgresql.org/download/)
* [Postgres & Django Interop](https://docs.djangoproject.com/en/3.2/ref/databases/#postgresql-notes)

#### Python
Python Requests module used to make REST API calls to various controllers, ingest replies in JSON encoded format.
From the returned JSON / Python Dictionary, use the Django database abstraction API to write returned JSON into a table hosted on a Postgres SQL database.

The django Webframework is also written in Python.

We'll step through defining our django models, ingesting REST-API output from controller and cloud APIs into the django models, and verifying table entries using PgAdmin to ultimately creating Dashboards such as below:

At the end of this series, we'll have stepped through creating basic dashboards and visualisations for:
* Meraki Dashboard
* Cisco SDWAN (vManage)
* Cisco ACI
* DCNM VXLAN and EVPN
* Cisco DNAC
* Openvuln API for PSIRT reporting


##### Meraki Devices:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki.Devices.png?raw=true)


##### DNAC Devices:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Screen%20Shot%202021-04-27%20at%209.23.26%20AM.png?raw=true)


##### Software Defined WAN Use Case
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/SDWAN.png?raw=true)


##### Openvuln API Dashboard
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/openvuln.png?raw=true)
