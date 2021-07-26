---
layout: default
title: 6.0 Cisco DNAC Dashboard
parent: Grafana Django and Cisco APIs Dashboard
nav_order: 9
last_modified_date: 2021-05-26 08:00:00 +1200
---


# 6.1 Introduction
This week we integrate DNA Center into our Dashboard for visibility into Campus Wired and Wireless domains.

Unlike DCNM and SD-WAN, there's not yet a supported or documented way to virtualise the DNA-C appliance (if you know of one, let me know!).  So instead of building a virtual lab, we'll use the DNA-C sandbox on DevNet and spend a little bit of time that we've saved on using Markdown language to supplement and customise our Dashboard.

By the end of this article we'll have created a dashboard that provides a single, aggregated view of:
* Connected Platforms
* Running Software Versions
* Device Roles and reachability
* An Aggregated view of site health over time and issues
* Summary and detailed view of Interface status
* Text Panels and Markdown syntax

Here's an example of the dashboard that we'll step through.

![DNA Dashboard](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/dna_dash-1.png?raw=true)


# 6.2 References
*  [DNA Center on DevNet](https://developer.cisco.com/dnacenter/)
*  [DNA-C API Reference Guide](https://developer.cisco.com/docs/dna-center/#!cisco-dna-center-platform-overview)
*  [DNAC GitHub Repo](https://github.com/CiscoDevNet/dnac-apis-with-python-sample-codes)
* [Markdown Language - Getting Started](https://www.markdownguide.org/getting-started/)

---
# 6.3 Create our DNAC App in Django

Create the DNA-C directory structure.
```shell
(venv) % python manage.py startapp dnac
```

If you've followed along with previous articles, we'll now have directories for dcnm, dnac, meraki, openvuln and sdwan.
```shell
.
├── apps
│   ├── dcnm_api.py
│   ├── dnac_api.py
│   ├── meraki_api.py
│   ├── openvuln_api.py
│   └── sdwan_api.py
├── cisco_grafana
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
├── dcnm
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── dnac
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
├── meraki
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── openvuln
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   ├── models.py
│   ├── tests.py
│   └── views.py
└── sdwan
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── migrations
    ├── models.py
    ├── tests.py
    └── views.py
```

Add the dnac app to Django's **settings.py**
```python
# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'meraki', # Added in previous post
    'dcnm', # Added in previous post
    'sdwan', # Added in previous post
    'openvuln', # Added in previous post
    'dnac', # < - Add this
]
```

---
# 6.4 Python Requests Functions

The [DNA Center Learning track](https://developer.cisco.com/learning/tracks/dnacenter-programmability/dnac-rest-apis/dnac-101-auth/step/2) has structured labs and sample code that I've used below to get you started.  I recommend stepping through these labs as a first step.

Create the  `Apps > dnac_api.py` file - this will store all our DNA-C python functions.

## 6.4.1 Retrieving the access token

Create the function below to retrieve the token that will be used to authenticate API calls.
```python
def get_auth_token(DNAC_USER='devnetuser', DNAC_PASSWORD='Cisco123!',
                    dnac='sandboxdnac.cisco.com'):
    """
    Building out Auth request. Using requests.post to make a call to the Auth Endpoint
    """
    url = 'https://{}/dna/system/api/v1/auth/token'.format(dnac)       # Endpoint URL
    resp = requests.post(url, auth=HTTPBasicAuth(DNAC_USER, DNAC_PASSWORD))  # Make the POST Request
    token = resp.json()['Token']    # Retrieve the Token from the returned JSON
    return token    # Create a return statement to send the token back for later use
```

## 6.4.2 Retrieve a list of devices registered with DNA-C

The function below will pass the token retrieved previously into the HTTP header when querying the API endpoint and return the API output in JSON format.

```python
def dnac_http_get(dnac_class):
    token = get_auth_token() # Get a Token
    url = "https://sandboxdnac.cisco.com/{}".format(dnac_class) #Network Device endpoint
    hdr = {'x-auth-token': token, 'content-type' : 'application/json'} #Build header Info
    #querystring = {"macAddress":"00:c8:8b:80:bb:00","managementIpAddress":"10.10.22.74"}
    querystring = ''

    resp = requests.get(url, headers=hdr, params=querystring)  # Make the Get Request
    device_list = resp.json() # Capture data from the controller
    return device_list # Pretty print the data

devices = dnac_http_get(dnac_class = 'api/v1/network-device')
pp.pprint(devices)
```

You might want to have a read of the [DNA-C API Reference](https://developer.cisco.com/docs/dna-center/#!cisco-dna-center-platform-overview), so that your familiar with the output to expect.

Run the function and check the output:
```shell
(venv)$ python dnac_api.py
{'response': [{'apEthernetMacAddress': None,
               'apManagerInterfaceIp': '',
               'associatedWlcIp': '',
               'bootDateTime': '2021-03-29 21:25:08',
               'collectionInterval': 'Global Default',
               'collectionStatus': 'Managed',
               'description': 'Cisco IOS Software [Bengaluru], ASR1000 '
                              'Software '
                              '(X86_64_LINUX_IOSD-UNIVERSALK9_NOLI-M), Version '
                              '17.4.1b, RELEASE SOFTWARE (fc2) Technical '
                              'Support: http://www.cisco.com/techsupport '
                              'Copyright (c) 1986-2021 by Cisco Systems, Inc. '
                              'Compiled Sat 06-Feb-21 0',
               'deviceSupportLevel': 'Supported',
               'errorCode': None,
               'errorDescription': None,
               'family': 'Routers',
               'hostname': 'asr1001-x.abc.inc',
               'id': '6aad2ec7-d1d0-4605-bf32-f62266c5f53e',
               'instanceTenantId': '602bebe514710a00c98fa402',
               'instanceUuid': '6aad2ec7-d1d0-4605-bf32-f62266c5f53e',
               'interfaceCount': '0',
               'inventoryStatusDetail': '<status><general '
                                        'code="SUCCESS"/></status>',
               'lastUpdateTime': 1621817528359,
               'lastUpdated': '2021-05-24 00:52:08',
               'lineCardCount': '0',
               'lineCardId': '',
               'location': None,
               'locationName': None,
               'macAddress': '00:c8:8b:80:bb:00',
               'managedAtleastOnce': True,
               'managementIpAddress': '10.10.22.253',
               'managementState': 'Managed',
               'memorySize': 'NA',
               'platformId': 'ASR1001-X',
               'reachabilityFailureReason': '',
               'reachabilityStatus': 'Reachable',
               'role': 'BORDER ROUTER',
               'roleSource': 'MANUAL',
               'serialNumber': 'FXS1932Q1SE',
               'series': 'Cisco ASR 1000 Series Aggregation Services Routers',
               'snmpContact': '',
               'snmpLocation': '',
               'softwareType': 'IOS-XE',
               'softwareVersion': '17.4.1b',
               'tagCount': '0',
               'tunnelUdpPort': None,
               'type': 'Cisco ASR 1001-X Router',
               'upTime': '55 days, 3:27:39.65',
               'uptimeSeconds': 4772142,
               'waasDeviceMode': None},

```

---

# 6.5 Django Models file and Database Definition
As in previous articles, we'll map the JSON output from our API calls into our Django Models files, which in turn writes our API responses to the postgres database.

Edit our **dnac > models.py** file / database definition and create below models to store the output of our API requests, we'll create models for:
* Devices associated with DNA-C
* Network health
* Network issues
* Interface details


```python
from django.db import models
import datetime
from django.utils import timezone

# Create your models here.
class dnac_network_device(models.Model):
    dnac_addr = models.CharField(max_length=200)
    mgt_ip = models.CharField(max_length=200)
    assoc_wlc = models.CharField(max_length=200)
    up_time_sec = models.BigIntegerField()
    device_support_level = models.CharField(max_length=200)
    sw_type = models.CharField(max_length=200)
    sw_ver = models.CharField(max_length=200)
    family = models.CharField(max_length=200)
    interface_count = models.CharField(max_length=200)
    line_card_count = models.CharField(max_length=200)
    platform_id = models.CharField(max_length=200)
    ser_number = models.CharField(primary_key=True, max_length=200)
    role = models.CharField(max_length=200)
    instance_tenant_id = models.CharField(max_length=200)
    id = models.CharField(max_length=200)
    reachability_status = models.CharField(max_length=200)
    manageability = models.CharField(max_length=200)
    mac = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.ser_number


class dnac_site_health(models.Model):
    dnac_addr = models.CharField(max_length=200)
    siteName = models.CharField(max_length=200)
    siteId = models.CharField(max_length=200)
    parentSiteId = models.CharField(max_length=200, null=True)
    siteType = models.CharField(max_length=200, null=True)
    healthyNetworkDevicePercentage = models.IntegerField(null=True)
    clientHealthWired = models.IntegerField(null=True)
    clientHealthWireless = models.IntegerField(null=True)
    numberOfClients = models.IntegerField(null=True)
    numberOfNetworkDevice = models.IntegerField(null=True)
    networkHealthAverage = models.IntegerField(null=True)
    networkHealthAccess = models.IntegerField(null=True)
    networkHealthCore = models.IntegerField(null=True)
    networkHealthDistribution = models.IntegerField(null=True)
    networkHealthRouter = models.IntegerField(null=True)
    networkHealthAP = models.IntegerField(null=True)
    networkHealthWLC = models.IntegerField(null=True)
    networkHealthSwitch = models.IntegerField(null=True)
    networkHealthWireless = models.IntegerField(null=True)
    networkHealthOthers = models.IntegerField(null=True)
    numberOfWiredClients = models.IntegerField(null=True)
    numberOfWirelessClients = models.IntegerField(null=True)
    wiredGoodClients = models.IntegerField(null=True)
    wirelessGoodClients = models.IntegerField(null=True)
    overallGoodDevices = models.IntegerField(null=True)
    accessGoodCount = models.IntegerField(null=True)
    accessTotalCount = models.IntegerField(null=True)
    coreGoodCount = models.IntegerField(null=True)
    coreTotalCount = models.IntegerField(null=True)
    distributionGoodCount = models.IntegerField(null=True)
    distributionTotalCount = models.IntegerField(null=True)
    routerGoodCount = models.IntegerField(null=True)
    routerTotalCount = models.IntegerField(null=True)
    wirelessDeviceGoodCount = models.IntegerField(null=True)
    wirelessDeviceTotalCount = models.IntegerField(null=True)
    apDeviceGoodCount = models.IntegerField(null=True)
    apDeviceTotalCount = models.IntegerField(null=True)
    wlcDeviceGoodCount = models.IntegerField(null=True)
    wlcDeviceTotalCount = models.IntegerField(null=True)
    switchDeviceGoodCount = models.IntegerField(null=True)
    switchDeviceTotalCount = models.IntegerField(null=True)
    applicationHealth = models.IntegerField(null=True)
    applicationGoodCount = models.IntegerField(null=True)
    applicationTotalCount = models.IntegerField(null=True)
    applicationBytesTotalCount = models.IntegerField(null=True)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.siteId

class dnac_issues(models.Model):
    dnac_addr = models.CharField(max_length=200)
    issueId = models.CharField(max_length=200)
    name = models.CharField(max_length=200)
    siteId = models.CharField(max_length=200)
    deviceId = models.CharField(max_length=200)
    deviceRole = models.CharField(max_length=200)
    aiDriven = models.CharField(max_length=200)
    clientMac = models.CharField(max_length=200)
    issue_occurence_count = models.IntegerField( null=True)
    status = models.CharField(max_length=200)
    priority = models.CharField(max_length=200)
    category = models.CharField(max_length=200)
    last_occurence_time = models.DateTimeField(primary_key=True, max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.issueId

class dnac_interfaces(models.Model):
    adminStatus = models.CharField(max_length=200)
    className = models.CharField(max_length=200)
    description = models.CharField(max_length=200)
    deviceId = models.CharField(max_length=200)
    duplex = models.CharField(max_length=200, null=True)
    int_id = models.CharField(max_length=200)
    ifIndex = models.CharField(max_length=200)
    instanceTenantId = models.CharField(max_length=200)
    instanceUuid = models.CharField(max_length=200)
    interfaceType = models.CharField(max_length=200)
    ipv4Address = models.CharField(max_length=200, null=True)
    ipv4Mask = models.CharField(max_length=200, null=True)
    lastUpdated = models.CharField(max_length=200)
    macAddress = models.CharField(max_length=200, null=True)
    mappedPhysicalInterfaceId = models.CharField(max_length=200, null=True)
    mappedPhysicalInterfaceName = models.CharField(max_length=200, null=True)
    mtu = models.CharField(max_length=200)
    nativeVlanId = models.CharField(max_length=200, null=True)
    isisSupport = models.CharField(max_length=200, null=True)
    ospfSupport = models.CharField(max_length=200, null=True)
    owningEntityId = models.CharField(max_length=200)
    pid = models.CharField(max_length=200)
    portMode = models.CharField(max_length=200)
    portName = models.CharField(max_length=200)
    portType = models.CharField(max_length=200)
    poweroverethernet = models.CharField(max_length=200)
    serialNo = models.CharField(max_length=200)
    speed = models.CharField(max_length=200)
    status = models.CharField(max_length=200)
    vlanId = models.CharField(max_length=200, null=True)
    voiceVlan = models.CharField(max_length=200, null=True)
    ser_port = models.CharField(primary_key=True, max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.ser_port

```
Commit the model to postgres sql:

```shell
(venv) jamespsullivan@JASULLI2-M-C0T0 cisco_grafana % python manage.py makemigrations dnac
Migrations for 'dnac':
  dnac/migrations/0001_initial.py
    - Create model dnac_issues
    - Create model dnac_network_device
    - Create model dnac_site_health
(venv) jamespsullivan@JASULLI2-M-C0T0 cisco_grafana % python manage.py migrate dnac       
Operations to perform:
  Apply all migrations: dnac
```

You can validate with pgAdmin the tables and column headers have been created in postgres.

---
# 6.6 Writing to Database
Create the below function to write our dnac request output to database using Django's database abstraction API:
```python
def get_dnac_network_device(query_class):
      try:
          devices = dnac_http_get(query_class)
          for device in devices['response']:
              device_entry = dnac_network_device(dnac_addr = dnac,
                  mgt_ip = device['managementIpAddress'],
                  assoc_wlc = device['associatedWlcIp'],
                  up_time_sec = device['uptimeSeconds'],
                  device_support_level = device['deviceSupportLevel'],
                  sw_type = device['softwareType'],
                  sw_ver = device['softwareVersion'],
                  family = device['family'],
                  interface_count = device['interfaceCount'],
                  line_card_count = device['lineCardCount'],
                  platform_id = device['platformId'],
                  ser_number = device['serialNumber'],
                  role = device['role'],
                  instance_tenant_id = device['instanceTenantId'],
                  id = device['id'],
                  reachability_status = device['reachabilityStatus'],
                  manageability= device['managementState'],
                  mac= device['macAddress'],
                  )
              device_entry.save()
          aci_model_pruner(dnac_network_device.objects.all(), obj_type="DNAC Device", retention_period=30)
      except:
          print('Unable to query devices on DNAC {}'.format(dnac))

def get_dnac_site_health(query_class):
    try:
        sites = dnac_http_get(query_class)
        for site in sites['response']:
            site_entry= dnac_site_health(
            dnac_addr =dnac,
            siteName = site['siteName'],
            siteId = site['siteId'],
            parentSiteId = site['parentSiteId'],
            siteType = site['siteType'],
            healthyNetworkDevicePercentage = site['healthyNetworkDevicePercentage'],
            clientHealthWired = site['clientHealthWired'],
            clientHealthWireless = site['clientHealthWireless'],
            numberOfClients = site['numberOfClients'],
            numberOfNetworkDevice = site['numberOfNetworkDevice'],
            networkHealthAverage = site['networkHealthAverage'],
            networkHealthAccess = site['networkHealthAccess'],
            networkHealthCore = site['networkHealthCore'],
            networkHealthDistribution = site['networkHealthDistribution'],
            networkHealthRouter = site['networkHealthRouter'],
            networkHealthAP = site['networkHealthAP'],
            networkHealthWLC = site['networkHealthWLC'],
            networkHealthSwitch = site['networkHealthSwitch'],
            networkHealthWireless = site['networkHealthWireless'],
            networkHealthOthers = site['networkHealthOthers'],
            numberOfWiredClients = site['numberOfWiredClients'],
            numberOfWirelessClients = site['numberOfWirelessClients'],
            wiredGoodClients = site['wiredGoodClients'],
            wirelessGoodClients = site['wirelessGoodClients'],
            overallGoodDevices = site['overallGoodDevices'],
            accessGoodCount = site['accessGoodCount'],
            accessTotalCount = site['accessTotalCount'],
            coreGoodCount = site['coreGoodCount'],
            coreTotalCount = site['coreTotalCount'],
            distributionGoodCount = site['distributionGoodCount'],
            distributionTotalCount = site['distributionTotalCount'],
            routerGoodCount = site['routerGoodCount'],
            routerTotalCount = site['routerTotalCount'],
            wirelessDeviceTotalCount = site['wirelessDeviceTotalCount'],
            apDeviceGoodCount = site['apDeviceGoodCount'],
            apDeviceTotalCount = site['apDeviceTotalCount'],
            wlcDeviceGoodCount = site['wlcDeviceGoodCount'],
            wlcDeviceTotalCount = site['wlcDeviceTotalCount'],
            switchDeviceGoodCount = site['switchDeviceGoodCount'],
            switchDeviceTotalCount = site['switchDeviceTotalCount'],
            applicationHealth = site['applicationHealth'],
            applicationGoodCount = site['applicationGoodCount'],
            applicationTotalCount = site['applicationTotalCount'],
            applicationBytesTotalCount = site['applicationBytesTotalCount'],
            )
        site_entry.save()

    except:
        print('Unable to query site health on DNAC {}'.format(dnac))

def get_dnac_issues(query_class):
  try:
      issues = dnac_http_get(query_class)
      for issue in issues['response']:
          dateTime = datetime.datetime.fromtimestamp(int(issue['last_occurence_time']/1000))
          converted_date = dateTime.strftime("%Y-%m-%d %H:%M:%S")
          issues_entry = dnac_issues(
          dnac_addr = dnac,
          issueId = issue['issueId'],
          name = issue['name'],
          siteId = issue['siteId'],
          deviceId = issue['deviceId'],
          deviceRole = issue['deviceRole'],
          aiDriven = issue['aiDriven'],
          clientMac = issue['clientMac'],
          issue_occurence_count = issue['issue_occurence_count'],
          status = issue['status'],
          priority = issue['priority'],
          category = issue['category'],
          last_occurence_time = str(converted_date)
          )
          issues_entry.save()
    except:
        print('Unable to query devices on DNAC {}'.format(dnac))

def get_dnac_interfaces(query_class):
  try:
    interfaces = dnac_http_get(query_class)
    for interface in interfaces['response']:
        interface_entry = dnac_interfaces(
        adminStatus = interface['adminStatus'],
        className = interface['className'],
        description = interface['description'],
        deviceId = interface['deviceId'],
        duplex = interface['duplex'],
        int_id = interface['id'],
        ifIndex = interface['ifIndex'],
        instanceTenantId = interface['instanceTenantId'],
        instanceUuid = interface['instanceUuid'],
        interfaceType = interface['interfaceType'],
        ipv4Address = interface['ipv4Address'],
        ipv4Mask = interface['ipv4Mask'],
        lastUpdated = interface['lastUpdated'],
        macAddress = interface['macAddress'],
        mappedPhysicalInterfaceId = interface['mappedPhysicalInterfaceId'],
        mappedPhysicalInterfaceName = interface['mappedPhysicalInterfaceName'],
        mtu = interface['mtu'],
        nativeVlanId = interface['nativeVlanId'],
        isisSupport = interface['isisSupport'],
        ospfSupport = interface['ospfSupport'],
        owningEntityId = interface['owningEntityId'],
        pid = interface['pid'],
        portName = interface['portName'],
        portType = interface['portType'],
        poweroverethernet = interface['poweroverethernet'],
        serialNo = interface['serialNo'],
        speed = interface['speed'],
        status = interface['status'],
        vlanId = interface['vlanId'],
        voiceVlan = interface['voiceVlan'],
        ser_port = interface['serialNo'] + '_' + interface['portName'],
        )
        interface_entry.save()
  except:
      print('Unable to query devices on DNAC {}'.format(dnac))

get_dnac_issues('dna/intent/api/v1/issues')
get_dnac_site_health('dna/intent/api/v1/site-health')
get_dnac_network_device('api/v1/network-device')
get_dnac_interfaces('api/v1/interface')

```
Call the functions `(venv) % python dnac_api.py` and check the output has been saved to database with PgAdmin:
![pgAdmin](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/dnac_pgAdmin.png?raw=true
)

---

# 6.7 Grafana

Let's create our dashboard to show, device and interface detail, issues, software versions, network health and text panels using markdown language.

* Create a new dashboard
* Add a new panel to show Network Health Score over time
* Time Column = **last_updated**
* Metric Column = **None**
* From the select column select **networkHealthAverage** & **healthyNetworkDevicePercentage**
* Select **Time Series** from the right
* Configure the table as needed

Your preview pane should now look like below - click save and apply
![Network Health](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DNA-network-health.png?raw=true)

* Add another panel
* Configure **query options** as below.  We'll query by the **PlatformId** field we created previously
* Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Repeat as needed for interfaces, software or other metrics your interested in - see the dashboard example at the top of this page for examples.

Your preview pane should now look like below - click save and apply
![Platform Overview](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/dna_platform_overview.png?raw=true)

Create a new panel to hold device detail as below
* From **dnac_dnac_network_device**
* Time Column = **last_updated**
* Add all relevant columns
* Format as table

Your preview pane should now look like below - click save and apply.
![Devices Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/dna_device_Table.png?raw=true)

Add other panels to your dashboard to suit your requirements.  There's some examples in the dashboard shown at the top of this page.

## 6.7.1 Alerting

We stepped through creating an alert for security advisories with the OpenVuln API in [update (5)](https://j-sulliman.github.io/2021/05/23/Part-5.0-OpenVuln-API-Cisco-PSIRT-and-Security-Vulnerability-Dashboard-with-Grafana.html).

In this example we create an alert if our network health score drops below 100%
![Health Alert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DNAC-Health-Alert.png?raw=true)

> Note: I've found by trial and error that alerting seems to work for  **Graph** type, but not time series or Pie Charts - We've changed our health time series graph above to a standard  **Graph** type.  The alert field is then configurable.  

Documentation on Grafana alerts can be found [here](https://grafana.com/docs/grafana/latest/alerting/create-alerts/) if you want to experiment further.

## 6.7.2 Markdown Language

Grafana supports the use of Markdown language in text boxes.  You can use this to add additional comments, links, or images like those in the example dashboard at the top of this page.

Markdown language is also used to format github **.md** files, on github pages (i.e. this page) and seems to be gaining popularity for technical documentation in general.

Here's a couple of examples from the dashboard:
* To insert an image `![Image Name](image_url)`
* To insert a hyperlink `[Link Text](link url)`
* To add headings #H1, ##H2, ###H3, etc

See Markdown documentation on GitHub [here](https://guides.github.com/features/mastering-markdown/).
