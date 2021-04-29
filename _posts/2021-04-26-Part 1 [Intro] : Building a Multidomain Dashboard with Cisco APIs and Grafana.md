---
layout: post
author: Jamie Sullivan
date:  2021-04-27 08:00:34 +1200
---
# Introduction - Network and Infra Monitoring with Grafana, Django and Python
In this series, we use REST APIs, Python, Django Webframework, PostgresSQL and Grafana to demonstrate building a cross-domain visualisations Dashboard for Cisco network and Infrastructure domains using REST APIs.

##### Lab Overview
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
* **Simple requirements**:  For a mock lab and self learning, Postgres SQL has more capability than I need.  For production environments you'll want to analyse requirements specific to your usecase and understand the pros and cons of timeseries databases such as **InfluxDB**, **prometheus** and **Logstash** for event/syslog aggregation.  

**Resources**
* [Postgres Installation](https://www.postgresql.org/download/)
* [Postgres & Django Interop](https://docs.djangoproject.com/en/3.2/ref/databases/#postgresql-notes)

#### Python
Python Requests module used to make REST API calls to various controllers, ingest replies in JSON encoded format.
From the returned JSON / Python Dictionary, use the Django database abstraction API to write returned JSON into a table hosted on a Postgres SQL database.

The django Webframework package is also written in Python.

We'll step through defining our django models, ingesting REST-API output from controller and cloud APIs into the django models, verifying table entries using PgAdmin and ultimately creating Dashboards such as below.

At the end of this series, we'll created basic dashboards and visualisations such as below for:
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

### Initial Setup

##### Create and activate a virtualenvironment and initialise an empty git repository
```python
zsh-shell Documents % mkdir grafana-cisco-demo             
zsh-shell Documents % cd grafana-cisco-demo
zsh-shell grafana-cisco-demo % virtualenv venv
zsh-shell grafana-cisco-demo % source venv/bin/activate
(venv) zsh-shell grafana-cisco-demo % git init
(venv) zsh-shell grafana-cisco-demo % nano .gitignore
^^ Add the venv/ directory to above .gitignore file
```

##### Install django
```python
(venv) zsh-shell grafana-cisco-demo % python -m pip install django
(venv) zsh-shell grafana-cisco-demo % python -m django --version
3.2
(venv) zsh-shell grafana-cisco-demo % cd cisco_grafana
(venv) zsh-shell cisco_grafana % python manage.py runserver
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

#### Install postgresql - or supported database of your choice

For installation options see - [Postgres Installation](https://www.postgresql.org/download/)

Once you've validated postgres's installed, edit your django settings file, so that django points to the postgres database.

```python
# Database
# https://docs.djangoproject.com/en/3.1/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'postgres',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}
```

Install python postgres sql package

```python
(venv) zsh-shell cisco_grafana % pip install psycopg2
```

### Wrap-up

We'll begin working with controller APIs, making requests, parsing responses, ingesting into our database and generating visualisations in the next update.

In the meantime - checkout DevNet if you haven't already. We'll re-use a lot of the code and examples shared in the [DevNet Learning Tracks](https://developer.cisco.com/startnow/) and the sandbox environments.

1. [x] This post - Introduction and Initial Setup
1. [ ] Next - Cisco Meraki Dashboard
   1. [ ] API Calls with Python Requests
   1. [ ] Commiting to databases
   1. [ ] Visualisation Database Tables with Grafana
