---
layout: post
author: Jamie Sullivan
date:  2021-04-27 08:00:34 +1200
---
# 2.0 Building a Meraki Dashboard with Grafana, Django and Python

## 2.1 Introduction
We continue from our overview and initial setup discussed in part (1), by exploring Meraki's APIs, adding functionality and models to our django install and visualising the captured data with Grafana.

| Section | Architecture | Link | Topic
------------ | ------------ | ------------- | -------------
1 | Introduction | [building a multidomain dashboard with cisco apis and grafana](https://j-sulliman.github.io/2021/04/26/Part-1-Intro-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Solution Overview
2 | Meraki | [Building a Meraki Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/04/26/Part-2-Meraki-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising Meraki APIs with Grafana.
3.1 | DCNM & EVPN VXLAN | [DCNM VXLAN BGP and EVPN Lab with Nexus 9000v](https://j-sulliman.github.io/2021/05/04/Part-3.1-DCNM-Lab-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | DCNM, N9000v, EVPN and VXLAN Lab deployment
3.2 | DCNM & EVPN VXLAN | [Building a DCNM Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/04/Part-3.2-DCNM-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising DCNM APIs with Grafana.
4.1 | SDWAN | [Cisco SDWAN Virtual Lab Build](https://j-sulliman.github.io/2021/05/13/Part-4.1-SDWAN-Cisco-SDWAN-Lab-Build.html) | Cisco SDWAN Virtual Lab deployment
4.2 | SDWAN | [Building a SD-WAN Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/17/Part-4.2-SDWAN-Cisco-SDWAN-vManage-API-Visualisations-with-Grafana.html) | Visualising vManage APIs with Grafana

## 2.2 Solution Overview - Cisco Meraki APIs and Visualisations with Grafana (green components)
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki_Solution_overview.png?raw=true)


## 2.3 Resources
* [Meraki Learning Track on DevNet](https://developer.cisco.com/startnow/#meraki-v0)
* [Postgres & Django Interop](https://docs.djangoproject.com/en/3.2/ref/databases/#postgresql-notes)

## 2.4 Implementation
### 2.4.1 Enable the Meraki API
From Dashboard, enable the API, under organisation settings as shown below, create and note down the API key from your profile
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Enable_meraki_api.png?raw=true)

###  2.4.2 Create the Meraki App (Directory Structure) in django
```python
zsh-shell grafana-cisco-demo % source venv/bin/activate
(venv) zsh-shell grafana-cisco-demo % cd cisco_grafana
(venv) zsh-shell cisco_grafana % python manage.py startapp meraki
(venv) zsh-shell cisco_grafana %
```
Django will have created a directory structure as shown:
```python
(venv) zsh-shell cisco_grafana % ls meraki
__init__.py	admin.py	apps.py		migrations	models.py	tests.py	views.py
```

Add the grafana app to cisco-grafana/settings.py:
```python
# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'meraki', ## Add this
]
```

### 2.4.3 Create Python API Requests functions to retrieve Meraki Orgs
```python

import requests
import time
import pprint as pp
from requests.auth import HTTPBasicAuth
requests.urllib3.disable_warnings(requests.urllib3.exceptions.InsecureRequestWarning)

def meraki_http_get(meraki_url):
    base_url = ('https://api.meraki.com/api/v1/{}'.format(meraki_url))

    headers = {'X-Cisco-Meraki-API-Key': '<your unique api key as defined above>'}
    try:
        get_response = requests.get(base_url, headers=headers, verify=False)
        status = get_response.status_code
        get_response = get_response.json()
        time.sleep(0.5)
    except:
        print('Meraki Cloud not reachable - check connection')
        get_response = 'unreachable'
        status = 'unreachable'
    return get_response, status

api_resp, api_status = meraki_http_get(meraki_url='organizations')
print(api_status)
pp.pprint(api_resp)
```
Run the function and validate the output.  You should receive a '200' response and your organisation id.  If not  (i.e. you receive an http response of 400), check that your API key and URL is correct - you can test the API call in Postman to validate that it's been enabled.

```python
(venv) zsh-shell apps % python meraki_api.py
200
[{'id': '863545',
  'name': 'Home',
  'url': 'https://n215.meraki.com/o/oDNX_a/manage/organization/overview'}]
```

At this point you should still have an empty 'models.py' file that was created when running __python manage.py createapp meraki__ command previously.  We'll insert our meraki model definitions into models.py below

```python
(venv) zsh-shell cisco_grafana % cat meraki/models.py
from django.db import models

# Create your models here.
```

Define a model/table to hold the organisation data that was returned above - edit the __meraki > models.py__ file so that it reflects below:

```python
from django.db import models
from django.utils import timezone

class Organizations(models.Model):
    id = models.CharField(max_length=200, primary_key=True)
    name = models.CharField(max_length=200)
    url = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)
```

Create the Organisations model using the makemigrations command as shown:
```python
(venv) zsh-shell cisco_grafana % python manage.py makemigrations meraki
Migrations for 'meraki':
  meraki/migrations/0001_initial.py
    - Create model Organizations
```

And commit the changes to the database:
```python
(venv) zsh-shell cisco_grafana % python manage.py migrate       
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, meraki, sessions
```

You can validate the table's been created with PgAdmin

### 2.4.4 Write output to database
Add below to the meraki_api.py file - see comments for explanation.

```python
import os
import sys

# Add your own path to django install location here
sys.path.append("/Users/jamespsullivan/Documents/grafana-cisco-demo/cisco_grafana/")

# We need to import django configuration settings to write our API response using the django models API
os.environ['DJANGO_SETTINGS_MODULE'] = 'cisco_grafana.settings'

import django
django.setup()

# Import the organisations model we defined earlier
from meraki.models import Organizations

# Write function to iterate over organisations found in our API request and save these to database
def meraki_orgs_to_db(data):
    for org in data:
        org_entry = Organizations(
            id = org['id'],
            name = org['name'],
            url = org['url']
            )
        org_entry.save()

# Call function and pass in our organisation data from api response
meraki_orgs_to_db(org_resp)
```
### 2.4.5 Retrieve meraki networks

Now that we have the meraki organisation data in the postgres database, we need to (a) Define a new model in meraki/models.py to hold network information, (b) iterate over each org entry in the database and send a REST API query to find configured networks:

#### 2.4.5.1 (a) Define a new model in meraki/models.py to hold network information

```python
class MerakiNetworks(models.Model):
    id = models.CharField(max_length=200, primary_key=True)
    name = models.CharField(max_length=200)
    organizationId = models.CharField(max_length=200)
    productTypes = models.CharField(max_length=200)
    timeZone = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)
```
Commit the new networks model to the database

```bash
(venv) zsh-shell cisco_grafana % python manage.py makemigrations meraki
Migrations for 'meraki':
  meraki/migrations/0002_merakinetworks.py
    - Create model MerakiNetworks
(venv) zsh-shell cisco_grafana % python manage.py migrate meraki
```
#### 2.4.5.2 (b) Iterate over each org entry in the database and send a REST API query to find configured networks
```python
from meraki.models import Organizations, MerakiNetworks # Edit this line to include MerakiNetworks model

def meraki_networks_to_db():
    for orgs in Organizations.objects.all():
        networks, status = meraki_http_get('organizations/{}/networks'.format(
                        orgs.id))
        for network in networks:
            network_entry = MerakiNetworks(
                id = network['id'],
                name = network['name'],
                organizationId = network['organizationId'],
                productTypes = network['productTypes'],
                timeZone = network['timeZone']
                )
            network_entry.save()
```

Validate the networks have been written to the database with PgAdmin
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki_PgAdmin.png?raw=true)


### 2.4.6 Retrieve Meraki Devices
We retrieved our network IDs in the previous step and can now use that network ID to run an API query for devices using the DN or syntax **/networks/{networkId}/devices**.

We can use the [Meraki API Documentation](https://developer.cisco.com/meraki/api-v1/#!get-network-devices), to create our model in meraki/models.py

### 2.4.7 MerakiDevices Model
```python
class MerakiDevices(models.Model):
    serial = models.CharField(max_length=200, primary_key=True)
    address = models.CharField(max_length=200)
    mac = models.CharField(max_length=200)
    lanIp = models.CharField(max_length=200, null=True)
    url = models.CharField(max_length=200)
    networkId = models.CharField(max_length=200)
    model = models.CharField(max_length=200)
    firmware = models.CharField(max_length=200)
    wan1Ip = models.CharField(max_length=200, null=True)
    wan2Ip = models.CharField(max_length=200, null=True)
    last_updated = models.DateTimeField(default=timezone.now)
```
Commit MerakiDevices model to database

```python
(venv) zsh-shell cisco_grafana % python manage.py makemigrations meraki
Migrations for 'meraki':
  meraki/migrations/0003_merakidevices.py
    - Create model MerakiDevices
(venv) zsh-shell cisco_grafana % python manage.py migrate meraki       
Operations to perform:
  Apply all migrations: meraki
```
```python
# Import the meraki devices model we defined previously
from meraki.models import Organizations, MerakiNetworks, MerakiDevices  

# Iterate over MerakiNetworks table and retrieve the NetworkID, use this
# to query networks for devices and write the output to MerakiDevices model we
# defined previously.
def meraki_devices_to_db():
    for networks in MerakiNetworks.objects.all():
        devices, status = meraki_http_get('networks/{}/devices'.format(
                        networks.id))
        for device in devices:
            if 'MX' in device['model']:
                device_entry = MerakiDevices(
                    address = device['address'],
                    serial = device['serial'],
                    mac = device['mac'],
                    lanIp = 'NA',
                    url = device['url'],
                    networkId = device['networkId'],
                    firmware = device['firmware'],
                    model = device['model'],
                    wan1Ip = device['wan1Ip'],
                    wan2Ip = device['wan2Ip']
                    )
                device_entry.save()
            elif 'MX' not in device['model']:
                device_entry = MerakiDevices(
                    address = device['address'],
                    serial = device['serial'],
                    firmware = device['firmware'],
                    mac = device['mac'],
                    lanIp = device['lanIp'],
                    url = device['url'],
                    networkId = device['networkId'],
                    model = device['model']
                    )
                device_entry.save()

meraki_devices_to_db()
```
Verify devices have been retrieved and written to database with PgAdmin.

# 2.5 Grafana Visualisations

## 2.5.1 Installation
Installation of Grafana is straightforward - you can find guides for Linux, macOS, Windows and containers
here [Grafana Installation](https://grafana.com/docs/grafana/latest/installation/)

Point Grafana to PostgresSQL as below
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Postgres%20datasource.png?raw=true)

Create a new dashboard and add a new panel - configure **query options** as below.  Note - if you don't configure __"aggregate: count"__ as shown, you'll most likely receive an error.

Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Your preview pane should now look like below - click save and apply
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/meraki_query.png?raw=true)

Return to the dashboard and duplicate the Devices PieChart, edit the duplicate chart, in the **query options**, replace **model** metric columns with **firmware**, you'll now have a summary of firmware breakdowns in the environment also.

* Return to the dashboard and add another panel, again select the **MerakiDevices** table.
* From the right hand pane, select **table**
* In the **query options**, select "format as: table"
* Add columns of interest to the table

![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki_device_table.png?raw=true)

If you've managed to follow the above successfully, your dashboard should look like below.
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/Meraki%20Dashboard%20Example.png?raw=true)


Using the [Meraki API Documentation](https://developer.cisco.com/meraki/api-v1/) as a guide, you can experiment with this process to create visualisations and graphs for other devices.

...follow up for DCNM dashboard to follow soon.

Jamie
