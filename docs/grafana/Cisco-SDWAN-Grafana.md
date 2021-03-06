---
layout: default
title: 4.2 Cisco SDWAN and vManage Dashboard
parent: Grafana Django and Cisco APIs Dashboard
nav_order: 7
last_modified_date: 2021-05-18 08:00:34 +1200
---


## 4.2.1 Overview

In parts two and three we covered creating Meraki and DCNM EVPN and VXLAN dashboards.   Last update (4.1), [Building an SDWAN Virtual Lab](https://j-sulliman.github.io/2021/05/13/Part-4.1-SDWAN-Cisco-SDWAN-Lab-Build.html), we created an SD-WAN virtual sandbox to to test our API calls and dashboards against.

This update we cover much the same process as previous updates but for SD-WAN and vManage:
* Running API calls against vManage NMS with Python
* Setting up django and database tables
* Ingesting the output of our vManage API call data into our Database using Django's database API
* Building our SD-WAN dashboard and visualisations with Grafana

![vManage and Grafana](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/vManage_Overview.png?raw=true)


## 4.2.2 Resources
* [vManage API Reference](https://developer.cisco.com/docs/sdwan/#!introduction)
* [SD-WAN learning tracks on DevNet](https://developer.cisco.com/sdwan/)

## 4.2.3 Django Setup
As in previous updates, we'll need to:
* Create our SD-WAN app in Django
* Create our model / table definitions to store the data from our API calls
* Map and save the data from our API calls to our database with the Django database API


### 4.2.3.1 Create the SD-WAN App in django

```python
(venv)% python manage.py startapp sdwan
```
If you've followed along with the previous meraki and DCNM articles, after running above, our directory structure should now look like this:
```
├── apps
│   ├── dcnm_api.py
│   └── meraki_api.py
├── cisco_grafana
│   ├── settings.py
│   ├── urls.py
├── db.sqlite3
├── dcnm
├── manage.py
├── meraki
└── sdwan
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── migrations
    │   └── __init__.py
    ├── models.py
    ├── tests.py
    └── views.py
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
    'meraki', # Added in previous article
    'dcnm', # Added in previous article
    'sdwan', # <- Add this
]
```
## 4.2.4 Making API calls to vManage with Python Requests

There's three steps to make an API call to vManage:
* Retrieve the XFSR (cross site forgery prevention token)
* Get the session id
* Make API requests passing in the token and session IDs

A detailed explanation as well as the sample code I've used can be found in the [vManage API Reference](https://developer.cisco.com/docs/sdwan/#!authentication/how-to-authenticate)

##### Create the session ID function in **apps > sdwan_api.py**:
```python
def get_jsessionid(vmanage_host, vmanage_port, username, password):
        api = "/j_security_check"
        base_url = "https://{}:{}".format(vmanage_host, vmanage_port)
        url = base_url + api
        payload = {'j_username' : username, 'j_password' : password}

        response = requests.post(url=url, data=payload, verify=False)
        try:
            cookies = response.headers["Set-Cookie"]
            jsessionid = cookies.split(";")
            return(jsessionid[0])
        except:
            if logger is not None:
                logger.error("No valid JSESSION ID returned\n")
            exit()
```
##### Call the function with your vmanage IP, username and password as inputs
```python
session_id = get_jsessionid(vmanage_host='192.168.200.42',
                vmanage_port='443',
                username='admin',
                password='Cisco123!')
print(session_id)
```

##### When **sdwan.py** is run, we should see the session ID as below:
```shell
(venv)% python sdwan_api.py
JSESSIONID=irE4-XUiUb3Z4bKyvNdO1I-J6fQqKX6y13cB9nXL.876a93d5-15e9-4d85-ae1c-6ecfed042886
```

##### Create the function to retreive the XFSR token:
```python
    def get_token(vmanage_host, vmanage_port, jsessionid):
        headers = {'Cookie': jsessionid}
        base_url = "https://{}:{}".format(vmanage_host, vmanage_port)
        api = "/dataservice/client/token"
        url = base_url + api
        response = requests.get(url=url, headers=headers, verify=False)
        if response.status_code == 200:
            return(response.text)
        else:
            return None
```
##### Call the **get_token()** function with host, port and the session_id as inputs.
```python
token = get_token(vmanage_host='192.168.200.42',
            vmanage_port='443',
            jsessionid=session_id)
print("Token: {}".format(token))
```
##### And verify that we've retrieved the token:
```shell
(venv)% python sdwan_api.py
JSESSIONID=z9l9e7e2_XNA3OtFtcN5vziq0BwAReFNQB_l0aqy.876a93d5-15e9-4d85-ae1c-6ecfed042886
Token: 06B2E640C58DB66335A94CF381E447BAEB6534972587362EF47BB3E1FF5023E8EA605FB09D57A288134FEFDE1B44BD717755
```

##### Create the function below.  It passes in the token and session ID into request headers and queries the API endpoint - see inline comments for explanation:
```python
def vmanage_get(token, jsessionid, vmanage_host='192.168.200.42',
    vmanage_port='443', querystring = '/devices'):
    # Pass our token and session-id to header used in requests query
    if token is not None:
        header = {'Content-Type': "application/json",'Cookie': jsessionid,
            'X-XSRF-TOKEN': token}
    else:
        header = {'Content-Type': "application/json",'Cookie': jsessionid}
    # The vmanage API service
    base_url = "https://{}:{}/dataservice".format(vmanage_host, vmanage_port)
    url = base_url + querystring
    # Pass the token and session ID to request headers
    response = requests.get(url=url, headers=header,verify=False)
    # Check that the request was successful
    if response.status_code == 200:
        try:
            items = response.json()['data']
        except KeyError:
            print('Data key does not exist')
            items = response.json()
        except TypeError:
            print('Type Error')
            print(response)
    elif response.status_code != 200:
        print('API Call Failed - received status code {}'.format(
            response.status_code))
        items = 'API Call Failed - received status code {}'.format(
            response.status_code)
    return items
```

##### Call the above function - we'll start with the **/device** API endpoint :
```python
devices = vmanage_get(token, session_id, vmanage_host='192.168.200.42',
                        vmanage_port='443',
                        querystring='/device'
```

##### When run, you'll receive a list of SD-WAN devices:
```python
'bfdSessions': '4',
  'bfdSessionsUp': 4,
  'board-serial': 'F38EC420',
  'certificate-validity': 'Valid',
  'connectedVManages': ['"1.1.1.1"'],
  'controlConnections': '2',
  'device-groups': ['"No groups"'],
  'device-model': 'vedge-cloud',
  'device-os': 'next',
  'device-type': 'vedge',
  'deviceId': '2.2.2.4',
  'domain-id': '1',
  'host-name': 'wgn-vedge-2',
  'isDeviceGeoData': False,
  'lastupdated': 1621218293544,
  'latitude': '37.666684',
  'layoutLevel': 4,
  'linux_cpu_count': '1',
  'local-system-ip': '2.2.2.4',
  'longitude': '-122.777023',
  'max-controllers': '0',
  'model_sku': 'None',
  'ompPeers': '1',
  'personality': 'vedge',
  'platform': 'x86_64',
  'reachability': 'reachable',
  'site-id': '2',
  'state': 'green',
  'state_description': 'All daemons up',
  'status': 'normal',
  'statusOrder': 4,
  'system-ip': '2.2.2.4',
  'testbed_mode': False,
  'timezone': 'Pacific/Auckland',
  'total_cpu_count': '2',
  'uptime-date': 1621218060000,
  'uuid': '12069106-eee8-08dc-4e99-83abc52e9e64',
  'validity': 'valid',
  'version': '20.3.3'}
```
### 4.2.5 Django Database API / Model Definition
Create the following database table entry in the the django models definitions -  **sdwan > models.py** - this will store devices retrieved from our API call to the postgres database:

```python
from django.db import models
from django.utils import timezone

# Create your models here.
class sdwan_devices(models.Model):
    deviceId = models.CharField(max_length=200)
    system_ip = models.CharField(max_length=200)
    host_name = models.CharField(max_length=200)
    reachability = models.CharField(max_length=200)
    status = models.CharField(max_length=200)
    personality = models.CharField(max_length=200)
    device_type = models.CharField(max_length=200)
    lastupdated = models.CharField(max_length=200)
    domain_id = models.CharField(max_length=200)
    board_serial = models.CharField(primary_key=True, max_length=200)
    certificate_validity = models.CharField(max_length=200)
    site_id = models.CharField(max_length=200)
    latitude = models.CharField(max_length=200)
    longitude = models.CharField(max_length=200)
    uptime_date = models.BigIntegerField()
    validity = models.CharField(max_length=200)
    state = models.CharField(max_length=200)
    state_description = models.CharField(max_length=200)
    local_system_ip = models.CharField(max_length=200)
    version = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.board_serial


class sdwan_deviceCounters(models.Model):
    crashCount = models.IntegerField()
    expectedControlConnections = models.IntegerField()
    number_vsmart_control_connections = models.IntegerField()
    rebootCount = models.IntegerField()
    systemIp = models.CharField(max_length=200)
    bfdSessionsUp = models.IntegerField(default=None)
    bfdSessionsDown = models.IntegerField(default=None)
    ompPeersDown = models.IntegerField(default=None)
    ompPeersUp = models.IntegerField(default=None)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.systemIp
```
Commit the model to postgres with the django makemigrations and migrate scripts:
```python
(venv)% python manage.py makemigrations sdwan
Migrations for 'sdwan':
  sdwan/migrations/0001_initial.py
    - Create model sdwan_devices

(venv)% python manage.py migrate sdwan       
Operations to perform:
  Apply all migrations: sdwan
```

### 4.2.6 Writing vManage API Calls to Database
Import the models into our **sdwan_api.py** script.  Using the function below we'll map the output of our devices API query into the sdwan_devices table using django's database API.  The vedges have different attributes returned than vmanage and vsmart:
```python
# Add the import statements torward the top of the file but below django.setup()
from sdwan.models import sdwan_devices, sdwan_deviceCounters

def devices_to_db(devices):
    for device in devices:
        if device['device-type'] == 'vmanage' or device['device-type'] == 'vsmart':
            sdwan_device = sdwan_devices(
            deviceId = device['deviceId'],
            system_ip = device['system-ip'],
            host_name = device['host-name'],
            reachability = device['reachability'],
            status = device['status'],
            personality = device['personality'],
            device_type = device['device-type'],
            lastupdated = device['lastupdated'],
            domain_id = device['domain-id'],
            board_serial = device['board-serial'],
            certificate_validity = device['certificate-validity'],
            site_id = device['site-id'],
            latitude = device['latitude'],
            longitude = device['longitude'],
            uptime_date = device['uptime-date'],
            validity = device['validity'],
            state = device['state'],
            state_description = device['state_description'],
            local_system_ip = device['local-system-ip'],
            version = device['version']
            )
            sdwan_device.save()
        elif device['device-type'] == 'vedge':
            sdwan_device = sdwan_devices(
            deviceId = device['deviceId'],
            system_ip = device['system-ip'],
            host_name = device['host-name'],
            reachability = device['reachability'],
            status = device['status'],
            personality = device['personality'],
            device_type = device['device-type'],
            lastupdated = device['lastupdated'],
            domain_id = device['domain-id'],
            board_serial = device['board-serial'],
            certificate_validity = device['certificate-validity'],
            site_id = device['site-id'],
            latitude = device['latitude'],
            longitude = device['longitude'],
            uptime_date = device['uptime-date'],
            validity = device['validity'],
            state = device['state'],
            state_description = device['state_description'],
            local_system_ip = device['local-system-ip'],
            version = device['version']
            )
            sdwan_device.save()
```

Call the function and pass in the list of devices:
```python
devices_to_db(devices)
```
Verify that the devices have been written to our postgresdb with PgAdmin:
![vManage PgAdmin](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/vmanage_pgadmin.png?raw=true)

##### Create a similar function, this time mapping the output of the API call to **/device/counters** to the models file/postgres Database.  Note: Again - you can view the parameters/expected output of the API call in the [vManage API Reference](https://developer.cisco.com/docs/sdwan/#!reference-device-state-and-statistics)
```python
def device_counter_to_db(device_counters):
    for counter in device_counters:
        print(counter)
        if 'bfdSessionsUp' in counter:
            device_counter = sdwan_deviceCounters(
            crashCount = counter['crashCount'],
            expectedControlConnections = counter['expectedControlConnections'],
            number_vsmart_control_connections = counter['number-vsmart-control-connections'],
            rebootCount = counter['rebootCount'],
            systemIp = counter['system-ip'],
            bfdSessionsUp = counter['bfdSessionsUp'],
            bfdSessionsDown = counter['bfdSessionsDown'],
            ompPeersDown = counter['ompPeersDown'],
            ompPeersUp = counter['ompPeersUp']
            )
            device_counter.save()
```
##### Call the functions:

```python
device_counters = vmanage_get(token, session_id, vmanage_host='192.168.200.42',
                        vmanage_port='443',
                        querystring='/device/counters'
                        )

device_counter_to_db(device_counters)
```
##### And validate that device counters have been saved to the database. You could use PgAdmin as before, or code similar to below to validate from python:

```python
for i in sdwan_deviceCounters.objects.all():
    print('-------------')
    print(i)
    print(i.bfdSessionsUp)
```

### 4.2.7 Grafana Visualisations

#### Installation
Refer to Tutuorial (2) Meraki for Grafana installation and setup instructions

* Create a new dashboard and add a new panel
* Configure **query options** as below.  Note - if you don't configure __"aggregate: count"__ as shown, you'll most likely receive an error.
* Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Your preview pane should now look like below - click save and apply
![Grafana Device Type](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/SDWAN_device_type.png?raw=true)

Repeat the same for any other values of interest (e.g. software version, reachability, status).

* Return to the dashboard and add another panel, again select the **sdwan_sdwan_devices** table.
* From the right hand pane, select **table**
* In the **query options**, select "format as: table"
* Add columns of interest to the table

![SD-WAN Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/SD-WAN_Table.png?raw=true)

* Return to the dashboard and add another panel, select the **sdwan_sdwan_devicecounters** table.
* From the right hand pane, select **time series**
* In the from options, Time Column = **last_updated**, metric column =  **systemIp**
* Select Column **bfdSessionsUp**

![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/BFD_sessions.png?raw=true)

Repeat for other counters of interest (for example OMP sessions).

Your SD-WAN Grafana Dashboard will look similar to below:

![SD-WAN Dashboard](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/SD-WAN_Dash.png?raw=true)


If you're still here, thank you for reading through.

I'll update this page with the next architecture soon.

Jamie
