---
layout: default
title: 7.0 Cisco ACI Dashboard
parent: Grafana Django and Cisco APIs Dashboard
nav_order: 10
last_modified_date: 2021-06-03 08:00:00 +1200
---

# 7.1 Introduction

Here we are at the seventh and final regular update to our series on building a multi-domain visualisations dashboard.

Through this series we've covered examples that demonstrate some of what is possible with Cisco API's to address the monitoring, visualisation and dashboard consolidation use case.  

This update we look into Cisco's software defined data center network solution, or ACI.  
![Solution Overview](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/ACI_Solution_overview.png?raw=true)

As mentioned in part one of this series, the nice thing about django, and python in general is its extensibility and flexibility to address future requirements as they arise.  

We'll add additional functionality by incorporating Django's send_mail function to provide customised, automated email alerting and the django_tables2 module to provide a web based view of all the tables stored in our database across all architectures.

We also:
* Explore the APICs APIs and build an ACI dashboard
* Bring the functions and code we've covered together into a single program (while loop) that will run in the background
* Provide consolidated dashboard examples - i.e. firmware and Performance
* Provide an example of Django's send mail function for email based notification when an object state changes

Here's an example of the dashboard for ACI that we'll step through.

![ACI Dashboard](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/ACI_Dash.png?raw=true)

Before we get to that though - a couple of sentences about what this grafana dashboard is, and what it isn't.  What we have shown this series, is a way to build a consolidated or "multi-domain" dashboard using API based monitoring, python and opensource software.

What we haven't shown is "Full Stack Observability", "user and application experience", for that here's a couple of primer discussions:
* [SDx Central article on Full Stack observability](https://www.sdxcentral.com/articles/news/cisco-melds-thousandeyes-appdynamics-for-full-stack-observability/2021/03/)
* [Application and User Experience on blogs.cisco.com](https://blogs.cisco.com/networking/improving-application-experience-with-full-stack-observability).

 > Both AppD and ThousandEyes, do have RESTful APIs, which means we could integrate feeds, metrics and data from both sources into our dashboard... (maybe in another update).

# 7.2 References

* [ACI on DevNet](https://developer.cisco.com/site/aci/)
* [ACI GitHub Repos](https://github.com/search?q=cisco+aci)
* [APIC REST API Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/2-x/rest_cfg/2_1_x/b_Cisco_APIC_REST_API_Configuration_Guide/b_Cisco_APIC_REST_API_Configuration_Guide_chapter_01.html)

# 7.3 Lab and Sandbox Environment

Cisco provides a free [APIC Simulator](https://software.cisco.com/download/home/285968390/type/286278832/release/4.2(7f)) that has a full featured API to test automation use cases against.  If you have access to an ESX or VMWare Workstation/Fusion host, this is an excellent tool that I've used extensively in the past.  If you prefer linux / KVM, until the Simulator's supported on hypervisor's other than VMWare, you'll need to use the always-on [sandbox](https://developer.cisco.com/docs/aci/#!sandbox/aci-sandboxes) APIC on devnet, or your own physical environment.

>  I've had some success with past simulator versions converting from a .vmdk disk image to a .qcow2 format using the [qemu-img convert tool](https://docs.openstack.org/image-guide/convert-images.html), I haven't been able to get this working with recent APIC simulator releases, the simulator powers on, startup script completes, however no management interface connectivity...interested to hear if anyone's managed to get this working.  Hopefully support for other hypervisors will be will be introduced in the future.

---
# 7.4 Implementation

# 7.4.1 Create our ACI App in Django

Create the ACI directory structure.
```shell
(venv) % python manage.py startapp aci
```

If you've followed along with previous articles, we'll now have directories for dcnm, dnac, meraki, openvuln, sdwan and ACI.
```shell
.
├── aci
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── apps
│   ├── dcnm_api.py
│   ├── dnac_api.py
│   ├── meraki_api.py
│   ├── openvuln_api.py
│   └── sdwan_api.py
├── cisco_grafana
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
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
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── openvuln
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
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

Add the aci app to Django's **settings.py**
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
    'dnac', # Added in previous post
    'aci', # < - Add this
]
```

---
# 7.4.2 Python Requests Functions

For consistency, we're going to continue using the python requests functions to make our API calls to the APIC controller.  There's a full featured "Cobra" SDK that's covered in the [Introduction to Cisco ACI Programmability](https://developer.cisco.com/site/aci/getting-started/) series if you prefer to use that instead. The SDK can be downloaded from [software.cisco.com](software.cisco.com), or the apic itself __http(s)://APIC address/cobra/_downloads/__

Create the  `Apps > aci_api.py` file - this will store all our ACI python functions.

## 7.4.2.1 Making an API call to the APIC

Create the function below to retrieve the token that will be used to authenticate API calls.
```python
def aci_get(mo_class, mgt = '' , usr = '', pw = ''):
    base_url = ('https://%s/api/' % mgt)
    cookies = {}
    name_pwd = {'aaaUser': {'attributes': {'name': usr, 'pwd': pw}}}
    json_credentials = json.dumps(name_pwd)

    # log in to API
    login_url = base_url + 'aaaLogin.json'
    post_response = requests.post(login_url, data=json_credentials, verify=False)
    if post_response.status_code != 200:
        print("API Request Failed for {}".format(mo_class))



    # get token from login response structure
    auth = json.loads(post_response.text)
    try:
        login_attributes = auth['imdata'][0]['aaaLogin']['attributes']
    except KeyError:
        print("Unable to retrieve logon attributes - wrong username or password?")
    auth_token = login_attributes['token']

    # create cookie array from token
    cookies['APIC-Cookie'] = auth_token
    sensor_url = (base_url + '/node/class/' + mo_class)
    get_response = requests.get(sensor_url, cookies=cookies, verify=False).json()
    return get_response
```

## 7.4.2.2 Retrieve a list of EPG's and Attached endpoints

Let's test the function against the API endpoint for EPGs.  The **rsp-subtree=full** string in the API call, will return all child objects of the EPG, including attached endpoints, associated bridge and VMM domains, path bindings.

```python
epgs = aci_get('fvAEPg.json?rsp-subtree=full', mgt='sandboxapicdc.cisco.com',
        usr='admin', pw='ciscopsdt')
pp.pprint(epgs)
```
Run the script from shell and check the output.  You can explore the API documentation for the APIC at the URL **https://apic-ip/doc/html/**, or the object browser **https://apic-ip/visore.html**
```shell
(venv) % python aci_api.py
{'fvAEPg': {'attributes': {'annotation': '',
                                       'childAction': '',
                                       'configIssues': '',
                                       'configSt': 'applied',
                                       'descr': '',
                                       'dn': 'uni/tn-SnV/ap-Power_Up/epg-Database',
                                       'exceptionTag': '',
                                       'extMngdBy': '',
                                       'floodOnEncap': 'disabled',
                                       'fwdCtrl': '',
                                       'hasMcastSource': 'no',
                                       'isAttrBasedEPg': 'no',
                                       'isSharedSrvMsiteEPg': 'no',
                                       'lcOwn': 'local',
                                       'matchT': 'AtleastOne',
                                       'modTs': '2021-06-03T02:35:58.130+00:00',
                                       'monPolDn': 'uni/tn-common/monepg-default',
                                       'name': 'Database',
                                       'nameAlias': '',
                                       'pcEnfPref': 'unenforced',
                                       'pcTag': '16392',
                                       'prefGrMemb': 'exclude',
                                       'prio': 'unspecified',
                                       'scope': '3047424',
                                       'shutdown': 'no',
                                       'status': '',
                                       'triggerSt': 'triggerable',
                                       'txId': '8070450532247928974',
                                       'uid': '15374'},
                        'children': [{'fvCEp': {'attributes': {'annotation': '',
                                                               'childAction': 'deleteNonPresent',
                                                               'contName': '',
                                                               'encap': 'vlan-128',
                                                               'extMngdBy': '',
                                                               'id': '0',
                                                               'idepdn': '',
                                                               'ip': '10.193.102.2',
                                                               'lcC': 'learned',
                                                               'lcOwn': 'local',
                                                               'mac': '44:CD:BB:C0:00:00',
                                                               'mcastAddr': 'not-applicable',
                                                               'modTs': '2021-06-03T02:39:32.589+00:00',
                                                               'monPolDn': 'uni/tn-common/monepg-default',
                                                               'name': '44:CD:BB:C0:00:00',
                                                               'nameAlias': '',
                                                               'rn': 'cep-44:CD:BB:C0:00:00',
                                                               'status': '',
                                                               'uid': '0',
                                                               'uuid': '',
                                                               'vmmSrc': ''},
                                                'children': [{'fvIp': {'attributes': {'addr': '10.193.102.2',
                                                                                      'annotation': '',
                                                                                      'childAction': 'deleteNonPresent',
                                                                                      'createTs': '1970-01-01T00:00:00.000+00:00',
                                                                                      'debugMACMessage': '',
                                                                                      'extMngdBy': '',
                                                                                      'lcOwn': 'local',
                                                                                      'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                      'monPolDn': 'uni/tn-common/monepg-default',
                                                                                      'rn': 'ip-[10.193.102.2]',
                                                                                      'status': '',
                                                                                      'uid': '0'},
                                                                       'children': [{'fvReportingNode': {'attributes': {'childAction': 'deleteNonPresent',
                                                                                                                        'createTs': '1970-01-01T00:00:00.000+00:00',
                                                                                                                        'id': '102',
                                                                                                                        'lcC': '',
                                                                                                                        'lcOwn': 'local',
                                                                                                                        'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                                                        'rn': 'node-102',
                                                                                                                        'status': ''}}}]}},
                                                             {'fvIp': {'attributes': {'addr': '2222::66:2',
                                                                                      'annotation': '',
                                                                                      'childAction': 'deleteNonPresent',
                                                                                      'createTs': '1970-01-01T00:00:00.000+00:00',
                                                                                      'debugMACMessage': '',
                                                                                      'extMngdBy': '',
                                                                                      'lcOwn': 'local',
                                                                                      'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                      'monPolDn': 'uni/tn-common/monepg-default',
                                                                                      'rn': 'ip-[2222::66:2]',
                                                                                      'status': '',
                                                                                      'uid': '0'},
                                                                       'children': [{'fvReportingNode': {'attributes': {'childAction': 'deleteNonPresent',
                                                                                                                        'createTs': '1970-01-01T00:00:00.000+00:00',
                                                                                                                        'id': '102',
                                                                                                                        'lcC': '',
                                                                                                                        'lcOwn': 'local',
                                                                                                                        'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                                                        'rn': 'node-102',
                                                                                                                        'status': ''}}}]}},
                                                             {'fvRsCEpToPathEp': {'attributes': {'childAction': 'deleteNonPresent',
                                                                                                 'forceResolve': 'yes',
                                                                                                 'lcC': 'learned',
                                                                                                 'lcOwn': 'local',
                                                                                                 'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                                 'rType': 'mo',
                                                                                                 'rn': 'rscEpToPathEp-[topology/pod-1/protpaths-101-102/pathep-[SnV_FI-1B]]',
                                                                                                 'state': 'formed',
                                                                                                 'stateQual': 'none',
                                                                                                 'status': '',
                                                                                                 'tCl': 'fabricAPathEp',
                                                                                                 'tDn': 'topology/pod-1/protpaths-101-102/pathep-[SnV_FI-1B]',
                                                                                                 'tType': 'mo'},
                                                                                  'children': [{'fvReportingNode': {'attributes': {'childAction': 'deleteNonPresent',
                                                                                                                                   'createTs': '1970-01-01T00:00:00.000+00:00',
                                                                                                                                   'id': '102',
                                                                                                                                   'lcC': 'learned',
                                                                                                                                   'lcOwn': 'local',
                                                                                                                                   'modTs': '2021-06-03T02:39:32.589+00:00',
                                                                                                                                   'rn': 'node-102',
                                                                                                                                   'status': ''}}}]}}]}},
```

# 7.4.3 Django Models file and Database Definition
Create the following model definition in the **aci > models.py** file - We'll go ahead and create definitions for other objects of interest while we're here:


```python
from django.db import models
import datetime
from django.utils import timezone

# Create your models here.
class aci_fvAEPg(models.Model):
    apic_addr = models.CharField(max_length=200)
    pcEnfPref = models.CharField(max_length=200)
    dn = models.CharField(primary_key=True, max_length=200)
    name = models.CharField(default='none', max_length=200)
    tenant = models.CharField(default='none', max_length=200)
    bd_tDn = models.CharField(max_length=200)
    fvRsDomAtt_tDn = models.CharField(max_length=200)
    fvRsPathAtt = models.CharField(max_length=200)
    fvRsCons = models.CharField(max_length=200)
    fvRsProv = models.CharField(max_length=200)
    modTs = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.apic_addr


class aci_firmwareRunning(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(primary_key=True, max_length=200)
    biosVer = models.CharField(max_length=200)
    version = models.CharField(max_length=200)
    descr = models.CharField(max_length=200)
    install_date = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.apic_addr

class aci_ethpmPhysIf(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(primary_key=True, max_length=200)
    node = models.CharField(max_length=200)
    operVlans = models.CharField(max_length=200)
    operMode = models.CharField(max_length=200)
    operSt = models.CharField(max_length=200)
    operSpeed = models.CharField(max_length=200)
    bundleIndex = models.CharField(max_length=200)
    backplaneMac = models.CharField(max_length=200)
    lastLinkStChg = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.apic_addr

class aci_eqptIngrTotal1d(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(max_length=200)
    node = models.CharField(max_length=200)
    bytesAvg = models.BigIntegerField()
    pktsAvg = models.CharField(max_length=200)
    utilAvg = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.apic_addr


class aci_fabricNodeHealth(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(max_length=200)
    healthLast = models.IntegerField()
    repIntvEnd = models.CharField(max_length=200)
    repIntvStart = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.apic_addr

class aci_lc_status(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(max_length=200)
    ser = models.CharField(primary_key=True, max_length=200)
    type = models.CharField(max_length=200)
    model = models.CharField(max_length=200)
    operSt = models.CharField(max_length=200)
    descr = models.CharField(max_length=200)
    upTs = models.CharField(max_length=200)
    manufactureTs = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.switch_name

class aci_cpu_status(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(max_length=200, default="null")
    idle_avg = models.FloatField(default=0)
    idle_min = models.FloatField(default=0)
    idle_max = models.FloatField(default=0)
    user_avg = models.FloatField(default=0)
    user_min = models.FloatField(default=0)
    user_max = models.FloatField(default=0)
    kern_avg = models.FloatField(default=0)
    kern_min = models.FloatField(default=0)
    kern_max = models.FloatField(default=0)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.switch_name

class aci_fabric_node(models.Model):
    apic_addr = models.CharField(max_length=200)
    dn = models.CharField(max_length=200)
    ser = models.CharField(primary_key=True, max_length=200)
    role = models.CharField(max_length=200)
    name = models.CharField(max_length=200)
    id = models.CharField(max_length=200)
    fabric_state = models.CharField(max_length=200, default="null")
    model = models.CharField(max_length=200)
    address = models.CharField(max_length=200)
    version = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.switch_name


class aci_bgpPeerEntry(models.Model):
    apic_addr = models.CharField(max_length=200)
    localIp = models.CharField(max_length=200)
    addr = models.CharField(max_length=200)
    holdIntvl = models.CharField(max_length=200)
    kaIntvl = models.CharField(max_length=200)
    dn = models.CharField(primary_key=True, max_length=200)
    vrf = models.CharField(max_length=200)
    operSt = models.CharField(max_length=200)
    passwdSet = models.CharField(max_length=200)
    rtrId = models.CharField(max_length=200)
    type = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)
    def __str__(self):
        return len(self.objects.all().filter(operSt='established'))


    def established(self):
        count = len(self.objects.all().filter(operSt='established'))
        return count


class aci_faultInst(models.Model):
    apic_addr = models.CharField(max_length=200)
    type = models.CharField(max_length=200)
    code = models.CharField(max_length=200)
    severity = models.CharField(max_length=200)
    subject = models.CharField(max_length=200)
    dn = models.CharField(primary_key=True, max_length=200)
    rule = models.CharField(max_length=200)
    descr = models.CharField(max_length=200)
    cause = models.CharField(max_length=200)
    lastTransition = models.DateTimeField(max_length=200)
    created = models.CharField(max_length=200)
    changeSet = models.CharField(max_length=200)
    ack = models.CharField(max_length=5)
    last_updated = models.DateTimeField(default=timezone.now)
    def __str__(self):
        return len(self.objects.all().filter(operSt='established'))


```
Commit the model to postgres sql:

```shell
(venv) % python manage.py makemigrations aci
Migrations for 'aci':
  aci/migrations/0001_initial.py
    - Create model aci_bgpPeerEntry
    - Create model aci_cpu_status
    - Create model aci_eqptIngrTotal1d
    - Create model aci_ethpmPhysIf
    - Create model aci_fabric_node
    - Create model aci_fabricNodeHealth
    - Create model aci_faultInst
    - Create model aci_firmwareRunning
    - Create model aci_fvAEPg
    - Create model aci_lc_status

(venv) % python manage.py migrate aci       

Operations to perform:
  Apply all migrations: aci
```
You'll see the tables we've created for ACI in PgAdmin, and if you've followed along with previous updates, the 30+ tables created for Meraki, SDWAN, DCNM, DNAC, OpenVuln API.

![pgAdmin Final](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/final_tables.png?raw=true)

Test the API calls as follows, you can pretty-print the output to validate:
```python

bds = aci_get('fvBD.json?rsp-subtree=full', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

firmware = aci_get('firmwareRunning.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

nodes = aci_get('fabricNode.json', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

node_health = aci_get('fabricNodeHealth.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

line_cards = aci_get('eqptLC.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

cpu = aci_get('procSysCPU5min.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

bgp_peer = aci_get('bgpPeerEntry.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')

faults = aci_get('faultInst.json?query-target=self', mgt='APIC-IP',
        usr='admin', pw='Cisco123!')


```

# 7.4.4 Email Notifications with Django's send_mail module

In part 6, we explored grafana's alerting functions, while useful it had limitations in that it only supported bar/line graph types.

Django provides the send_mail module that allows email notifications to be sent from Django.  We'll use this to create alerting from our application if an object status changes, or configuration is modified.  

With the call home email alert functionality of ACI and other Cisco products, do we really need to use a seperate function to implement email based alerts? Short answer, no.  But if we wanted to include devices in our dashboard that don't support call home this could be useul..worst case, we have sample code in our library to use for another project later.

In the example below we send an alert if a configuration change is made to an object. Configurable objects in ACI have a modified time stamp attribute or **modTs**.  If this attribute changes between API calls, it would indicate that a configuration change has occurred.

Here a sample of the code we'll use - which is pretty much saying, if the modified time stamp in our current API call, is different to the modified time stamp in our previous API call, send an email notifying that a configuration change has occured:

```python
if prev_bd_state.modTs !=fvBD["fvBD"]['attributes']['modTs']:
        send_email(fromip=apic, faulttype="Configuration Change", dn=fvBD["fvBD"]['attributes']['dn'],
                   description=fvBD["fvBD"], timestamp=str(datetime.datetime.now()))
```

In this second example, we check the faultInst objects for recent updates based on the **lastTransition** field. If the fault is less than 1000 seconds old, send an email with the fault type, code, severity:

```python
time_data = fault["faultInst"]['attributes']['lastTransition']
            split_time_data = time_data.split("+")
            struct_time = datetime.datetime.strptime(split_time_data[0],
                                                     "%Y-%m-%dT%H:%M:%S.%f")
            time_now = datetime.datetime.now()
            time_diff = time_now - struct_time
            if time_diff.seconds < 1000:
                send_email(fromip="10.67.36.20", # < Edit to suite
                           faulttype=fault["faultInst"]['attributes']['type'],
                           code=fault["faultInst"]['attributes']['code'],
                           severity=fault["faultInst"]['attributes']['severity'],
                           dn=fault["faultInst"]['attributes']['dn'],
                           rule=fault["faultInst"]['attributes']['rule'],
                           description=fault["faultInst"]['attributes']['descr'],
                           cause=fault["faultInst"]['attributes']['cause'],
                           timestamp=fault["faultInst"]['attributes']['lastTransition'])
```
For the above functions to work, add similar to below to `settings.py`, in my case I'm using a personal gmail account:
```python
# Mail Settings
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = 'your-email@gmail.com'
EMAIL_HOST_PASSWORD = 'your-email-pw' #past the key or password app here
EMAIL_PORT = 587
EMAIL_USE_TLS = True
DEFAULT_FROM_EMAIL = 'cisco_grafana'
```

Create an `apps / notifications.py` file and add the following:
```python
from django.core.mail import send_mail

def send_email(
    fromip,
    faulttype,
    code='',
    severity='',
    dn='',
    rule='',
    description='',
    cause='',
    timestamp=''):

    data1 = 'From IP  Address: {} Type: {} Code: {} Severity: {} \n'.format(
    fromip, faulttype, code, severity)
    data2 = 'DN: {} Rule: {} Desc: {} Cause: {} Transition: {}\n'.format(
    dn, rule, description, cause, timestamp)

    send_mail(
        # Email subject
        'Fault Code {} Severity {} raised at {}'.format(
            code, severity, timestamp),
        # Message body
        data1 + data2,
        #From Address
        'your-email@gmail.com',
        #To Address
        ['your-to-address@gmail.com'],
        fail_silently=False)
```
And import the function towards the top of aci_api.py:
```python
from notifications import send_email
```
We'll use the send_email function below when comparing current API call data to previous API call entries in our database.

When run, we should receive email alerts as below when a new fault or configuration change is found:
![email alert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/email_alert.png?raw=true)

To read more on the send_mail module see - [Sending Email](https://docs.djangoproject.com/en/3.2/topics/email/).  

---
# 7.4.5 Writing to Database
Create the below function to write our request output to database using Django's database abstraction API:
```python
def aci_fabric_node_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    node_data = aci_get('fabricNode.json', mgt,
            user, pw)
    for node in node_data['imdata']:
        try:
            cdpIntfAddr =''
            s1 = node['fabricNode']['attributes']['dn']
            s2 = s1.split('node-')
            s3 = s2[1].split('/')

            node_entry = aci_fabric_node(apic_addr=mgt,
                    dn=node['fabricNode']['attributes']['dn'],
                    ser=node['fabricNode']['attributes']['serial'],
                    role=node['fabricNode']['attributes']['role'],
                    name=node['fabricNode']['attributes']['name'],
                    id=node['fabricNode']['attributes']['id'],
                    model=node['fabricNode']['attributes']['model'],
                    fabric_state=node['fabricNode']['attributes']['fabricSt'],
                    address=node['fabricNode']['attributes']['address'],
                    version=node['fabricNode']['attributes']['version'],
                    )
            node_entry.save()
        except:
            print('Unable to query Fabric Node on APIC {}'.format(mgt))


def aci_fvBD_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    fvBD_data = aci_get("fvBD.json?rsp-subtree=full", mgt, user, pw)
    for fvBD in fvBD_data['imdata']:
        #try:
            fvRsBDToOut1 = ''
            fvRsBDToOut2 = ''
            fvRsBDToOut3 = ''
            fvSubnet1 = ''
            fvSubnet2 = ''
            fvSubnet3 = ''
            fvSubnet1_scope = ''
            fvSubnet2_scope = ''
            fvSubnet3_scope = ''
            fvRsCtx = ''
            for fvbd_child in fvBD["fvBD"]["children"]:
                if "fvRsBDToOut" in fvbd_child and fvRsBDToOut1 == '':
                    fvRsBDToOut1 = fvbd_child['fvRsBDToOut']["attributes"]['tDn']
                elif "fvRsBDToOut" in fvbd_child and fvRsBDToOut2 == '':
                    fvRsBDToOut2 = fvbd_child['fvRsBDToOut']["attributes"]['tDn']
                elif "fvRsBDToOut" in fvbd_child and fvRsBDToOut3 == '':
                    fvRsBDToOut3 = fvbd_child['fvRsBDToOut']["attributes"]['tDn']
                elif "fvSubnet" in fvbd_child and fvSubnet1 == '':
                    fvSubnet1 = fvbd_child['fvSubnet']["attributes"]['ip']
                    fvSubnet1_scope = fvbd_child['fvSubnet']["attributes"]['scope']
                elif "fvSubnet" in fvbd_child and fvSubnet2 == '':
                    fvSubnet2 = fvbd_child['fvSubnet']["attributes"]['ip']
                    fvSubnet2_scope = fvbd_child['fvSubnet']["attributes"]['scope']
                elif "fvSubnet" in fvbd_child and fvSubnet3 == '':
                    fvSubnet3 = fvbd_child['fvSubnet']["attributes"]['ip']
                    fvSubnet3_scope = fvbd_child['fvSubnet']["attributes"]['scope']
                elif "fvRsCtx" in fvbd_child and fvRsCtx == '':
                    fvRsCtx = fvbd_child['fvRsCtx']['attributes']['tDn']
            #try:
                prev_bd_state = aci_fvBD.objects.get(dn=fvBD['fvBD']['attributes']['dn'])
                if prev_bd_state.modTs !=fvBD["fvBD"]['attributes']['modTs']:
                        send_email(fromip=mgt, faulttype="Configuration Change", dn=fvBD["fvBD"]['attributes']['dn'],
                                   description=fvBD["fvBD"], timestamp=str(datetime.datetime.now()))
            #except:
                #print("Query for existing Object not found - New object?")
            fvBD_entry = aci_fvBD(apic_addr=mgt,
                                  descr=fvBD["fvBD"]['attributes']['descr'],
                                  dn=fvBD["fvBD"]['attributes']['dn'],
                                  arpFlood=fvBD["fvBD"]['attributes']['arpFlood'],
                                  epMoveDetectMode=fvBD["fvBD"]['attributes']['epMoveDetectMode'],
                                  ipLearning=fvBD["fvBD"]['attributes']['ipLearning'],
                                  limitIpLearnToSubnets=fvBD["fvBD"]['attributes']['limitIpLearnToSubnets'],
                                  name=fvBD["fvBD"]['attributes']['name'],
                                  unicastRoute=fvBD["fvBD"]['attributes']['unicastRoute'],
                                  unkMacUcastAct=fvBD["fvBD"]['attributes']['unkMacUcastAct'],
                                  unkMcastAct=fvBD["fvBD"]['attributes']['unkMcastAct'],
                                  modTs=fvBD["fvBD"]['attributes']['modTs'],
                                  fvSubnet1=fvSubnet1, fvSubnet2=fvSubnet2, fvSubnet3=fvSubnet3,
                                  fvSubnet1_scope=fvSubnet1_scope, fvSubnet2_scope=fvSubnet2_scope,
                                  fvSubnet3_scope=fvSubnet3_scope, fvRsBDToOut1=fvRsBDToOut1, fvRsBDToOut2=fvRsBDToOut2,
                                  fvRsBDToOut3=fvRsBDToOut3, fvRsCtx=fvRsCtx
                                  )

            fvBD_entry.save()
        #except:
            #print('Unable to query Bridge Domain on APIC {}'.format(mgt))

def aci_fvAEPg_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    epg_data = aci_get('fvAEPg.json?rsp-subtree=full', mgt, user, pw)
    for epg in epg_data['imdata']:
        epg_dict = {'fvRsPathAtt': [], 'fvRsDomAtt': [], 'fvRsCons': [], 'fvRsProv': [], 'fvRsBd': []}
        tn_1 = epg['fvAEPg']['attributes']['dn'].split('-')
        tn_2 = tn_1[1].split('/')
        try:
            prev_epg_state = aci_fvAEPg.objects.get(dn=epg['fvAEPg']['attributes']['dn'])
        except:
            prev_epg_state = "null"
        for epg_child in epg["fvAEPg"]["children"]:
            if 'fvRsPathAtt' in epg_child:
                epg_dict['fvRsPathAtt'].append(epg_child['fvRsPathAtt']['attributes']['tDn'])
            elif 'fvRsDomAtt' in epg_child:
                epg_dict['fvRsDomAtt'].append(epg_child['fvRsDomAtt']['attributes']['tDn'])
            elif 'fvRsCons' in epg_child:
                epg_dict['fvRsCons'].append(epg_child['fvRsCons']['attributes']['tDn'])
            elif 'fvRsProv' in epg_child:
                epg_dict['fvRsProv'].append(epg_child['fvRsProv']['attributes']['tDn'])
            elif 'fvRsBd' in epg_child:
                epg_dict['fvRsBd'].append(epg_child['fvRsBd']['attributes']['tDn'])

        epg_entry = aci_fvAEPg(apic_addr=mgt, dn=epg['fvAEPg']['attributes']['dn'],
                               pcEnfPref=epg['fvAEPg']['attributes']['pcEnfPref'],
                               name=epg['fvAEPg']['attributes']['name'],
                               tenant=tn_2[0],
                               bd_tDn=epg_dict['fvRsBd'],
                               fvRsDomAtt_tDn=epg_dict['fvRsDomAtt'],
                               fvRsPathAtt=epg_dict['fvRsPathAtt'],
                               fvRsCons=epg_dict['fvRsCons'],
                               fvRsProv=epg_dict['fvRsProv'],
                               modTs=epg['fvAEPg']['attributes']['modTs'])
        if prev_epg_state != 'null' and prev_epg_state.modTs != epg['fvAEPg']['attributes']['modTs']:
            send_email(fromip=mgt, faulttype="Configuration Change", dn=epg['fvAEPg']['attributes']['dn'],
                       description=epg['fvAEPg'], timestamp=str(datetime.datetime.now()))
        epg_entry.save()

def aci_firmwareRunning_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    fw_data = aci_get("firmwareRunning.json?query-target=self", mgt, user, pw)
    for fw in fw_data['imdata']:
        try:
            fw_entry = aci_firmwareRunning(
                apic_addr=mgt,
               dn=fw["firmwareRunning"]['attributes']['dn'],
               biosVer=fw["firmwareRunning"]['attributes']['biosVer'],
               version=fw["firmwareRunning"]['attributes']['version'],
               descr=fw["firmwareRunning"]['attributes']['descr'],
               install_date=fw["firmwareRunning"]['attributes']['ts'])
            fw_entry.save()
        except:
            print('Unable to query firmwareRunning on APIC {}'.format(mgt))


def fabricNodeHealth(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    nodes = aci_get("fabricNodeHealth.json?query-target=self", mgt, user, pw)
    for node in nodes['imdata']:
        try:
            if 'fabricNodeHealth15min' in node.keys():
                node_entry = aci_fabricNodeHealth(
                   apic_addr=mgt,
                   dn=node["fabricNodeHealth15min"]['attributes']['dn'],
                   healthLast=node["fabricNodeHealth15min"]['attributes']['healthLast'],
                   repIntvEnd=node["fabricNodeHealth15min"]['attributes']['repIntvEnd'],
                   repIntvStart=node["fabricNodeHealth15min"]['attributes']['repIntvStart'])
                node_entry.save()
        except:
            print('Unable to query Node Firmware on APIC {}'.format(mgt))


def aci_eqptlc_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    lc_data = aci_get("eqptLC.json?query-target=self", mgt, user, pw)
    for lc in lc_data['imdata']:
        try:
            lc_entry = aci_lc_status(apic_addr=mgt,
                                    dn=lc["eqptLC"]['attributes']['dn'],
                                    ser=lc["eqptLC"]['attributes']['ser'],
                                    type=lc["eqptLC"]['attributes']['type'],
                                    model=lc["eqptLC"]['attributes']['model'],
                                    operSt=lc["eqptLC"]['attributes']['operSt'],
                                    descr = lc["eqptLC"]['attributes']['descr'],
                                     upTs=lc["eqptLC"]['attributes']['upTs'],
                                     manufactureTs=lc["eqptLC"]['attributes']['mfgTm']
                                     )
            lc_entry.save()
        except:
            print('Unable to query eqptLC on APIC {}'.format(mgt))


def aci_eqptingrtotal1d_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    intf_data = aci_get("eqptIngrTotal1d.json?query-target=self", mgt, user, pw)
    for intf in intf_data['imdata']:
        try:
            s1 = intf['eqptIngrTotal1d']['attributes']['dn']
            s2 = s1.split('node-')
            s3 = s2[1].split('/')
            intf_entry = aci_eqptIngrTotal1d(
                apic_addr=mgt,
                dn=intf["eqptIngrTotal1d"]['attributes']['dn'],
                node=s3[0],
                 pktsAvg=intf["eqptIngrTotal1d"]['attributes']['pktsAvg'],
                 utilAvg=intf["eqptIngrTotal1d"]['attributes']['utilAvg'],
                 bytesAvg=intf["eqptIngrTotal1d"]['attributes']['bytesAvg'])
            intf_entry.save()
        except:
            print('Unable to query eqptIngrTotal1d on APIC {}'.format(mgt))

def aci_cpu_status_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    cpu_data = aci_get("procSysCPU5min.json?query-target=self", mgt, user, pw)
    for cpu in cpu_data['imdata']:
        try:
            s1 = cpu['procSysCPU5min']['attributes']['dn']
            s2 = s1.split('node-')
            s3 = s2[1].split('/')
            cpu_entry = aci_cpu_status(
                apic_addr=mgt,
                dn=cpu["procSysCPU5min"]['attributes']['dn'],
                idle_avg=cpu["procSysCPU5min"]['attributes']['idleAvg'],
                idle_min=cpu["procSysCPU5min"]['attributes']['idleMin'],
                idle_max=cpu["procSysCPU5min"]['attributes']['idleMax'],
                user_avg=cpu["procSysCPU5min"]['attributes']['userAvg'],
                user_min=cpu["procSysCPU5min"]['attributes']['userMin'],
                user_max=cpu["procSysCPU5min"]['attributes']['userMax'],
                kern_avg=cpu["procSysCPU5min"]['attributes']['kernelAvg'],
                kern_min=cpu["procSysCPU5min"]['attributes']['kernelMin'],
                kern_max=cpu["procSysCPU5min"]['attributes']['kernelMax'])
            cpu_entry.save()
        except:
            print('Unable to query procSysCPU5min on APIC {}'.format(mgt))


def aci_bgpPeerEntry_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    aci_bgpPeerEntry_data = aci_get('bgpPeerEntry.json?query-target=self', mgt, user, pw)
    for peer in aci_bgpPeerEntry_data['imdata']:
        try:
            s1 = peer['bgpPeerEntry']['attributes']['dn']
            s2 = s1.split(':')
            try:
                s3 = s2[1].split('/')
                s4 = s3[0]
            except:
                s4 = 'overlay-1'
            aci_peer_entry = aci_bgpPeerEntry(
                apic_addr=mgt, dn=peer['bgpPeerEntry']['attributes']['dn'],
                localIp=peer['bgpPeerEntry']['attributes']['localIp'],
                addr=peer['bgpPeerEntry']['attributes']['addr'],
                rtrId=peer['bgpPeerEntry']['attributes']['rtrId'],
                vrf = s4,
                operSt=peer['bgpPeerEntry']['attributes']['operSt'],
                holdIntvl=peer['bgpPeerEntry']['attributes']['holdIntvl'],
                kaIntvl=peer['bgpPeerEntry']['attributes']['kaIntvl'],
                passwdSet=peer['bgpPeerEntry']['attributes']['passwdSet'],
                type=peer['bgpPeerEntry']['attributes']['type'])
            aci_peer_entry.save()
        except:
            print('Unable to query bgpPeerEntry on APIC {}'.format(mgt))

def aci_fault_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    fault_data = aci_get("faultInst.json?query-target=self", mgt, user, pw)
    for fault in fault_data['imdata']:
        try:
            aci_fault_entry = aci_faultInst(
                apic_addr=mgt,
                dn=fault["faultInst"]['attributes']['dn'],
                type=fault["faultInst"]['attributes']['type'],
                code=fault["faultInst"]['attributes']['code'],
                severity=fault["faultInst"]['attributes']['severity'],
                subject=fault["faultInst"]['attributes']['subject'],
                descr = fault["faultInst"]['attributes']['descr'],
                rule=fault["faultInst"]['attributes']['rule'],
                cause=fault["faultInst"]['attributes']['cause'],
                lastTransition=fault["faultInst"]['attributes']['lastTransition'],
                changeSet=fault["faultInst"]['attributes']['changeSet'],
                ack=fault["faultInst"]['attributes']['ack'])
            aci_fault_entry.save()
            time_data = fault["faultInst"]['attributes']['lastTransition']
            split_time_data = time_data.split("+")
            struct_time = datetime.datetime.strptime(split_time_data[0],
                                                     "%Y-%m-%dT%H:%M:%S.%f")
            time_now = datetime.datetime.now()
            time_diff = time_now - struct_time
            if time_diff.seconds < 1000:
                send_email(fromip="10.67.36.20",
                           faulttype=fault["faultInst"]['attributes']['type'],
                           code=fault["faultInst"]['attributes']['code'],
                           severity=fault["faultInst"]['attributes']['severity'],
                           dn=fault["faultInst"]['attributes']['dn'],
                           rule=fault["faultInst"]['attributes']['rule'],
                           description=fault["faultInst"]['attributes']['descr'],
                           cause=fault["faultInst"]['attributes']['cause'],
                           timestamp=fault["faultInst"]['attributes']['lastTransition'])
        except:
            print('Unable to query faultInst on APIC {}'.format(mgt))

def aci_ethpmphysif_update(mgt='10.67.36.20', user='admin', pw='Cisco123!'):
    phyif_data = aci_get("ethpmPhysIf.json?query-target=self", mgt, user, pw)
    for intf in phyif_data['imdata']:
        try:
            s1 = intf['ethpmPhysIf']['attributes']['dn']
            s2 = s1.split('node-')
            s3 = s2[1].split('/')
            aci_ethpmphysif_entry = aci_ethpmPhysIf(
                apic_addr=mgt,
                dn=intf["ethpmPhysIf"]['attributes']['dn'],
                node=s3[0],
                operSt=intf["ethpmPhysIf"]['attributes']['operSt'],
                operVlans=intf["ethpmPhysIf"]['attributes']['operVlans'],
                operMode=intf["ethpmPhysIf"]['attributes']['operMode'],
                operSpeed=intf["ethpmPhysIf"]['attributes']['operSpeed'],
                bundleIndex = intf["ethpmPhysIf"]['attributes']['bundleIndex'],
                backplaneMac=intf["ethpmPhysIf"]['attributes']['backplaneMac'],
                lastLinkStChg=intf["ethpmPhysIf"]['attributes']['lastLinkStChg'])
            aci_ethpmphysif_entry.save()
        except:
            print('Unable to query ethpmPhysIf on APIC {}'.format(mgt))
```
# 7.4.6 Aggregating and continuously running our functions

Until now, we've been calling our API functions arbitrarily from our scripts, this being the last update, we'll consolidate these into a single application via a While Loop so they can be aggregated and run continuously.

Create the file `cisco_grafana_main.py` in the root directory (the same directory level as `manage.py`) and add the following.

You can add the functions we created in previous articles here also.  I've shown an example below for meraki, dcnm, dnac and aci functions.


```python
from apps.aci_api import aci_fabric_node_update, aci_fvBD_update
from apps.aci_api import aci_fvAEPg_update, aci_firmwareRunning_update
from apps.aci_api import fabricNodeHealth, aci_eqptlc_update
from apps.aci_api import aci_eqptingrtotal1d_update, aci_cpu_status_update
from apps.aci_api import aci_bgpPeerEntry_update, aci_fault_update
from apps.aci_api import aci_ethpmphysif_update
from apps.meraki_api import meraki_http_get, meraki_orgs_to_db
from apps.meraki_api import meraki_devices_to_db, meraki_networks_to_db
from apps.dcnm_api import get_dcnm_network_device, get_dcnm_network_switch
from apps.dnac_api import get_dnac_network_device, get_dnac_issues
from apps.dnac_api import get_dnac_site_health, get_dnac_interfaces

while True:
    get_dcnm_network_switch('/inventory/switches')
    get_dcnm_network_device('/control/fabrics')
    get_dnac_issues('dna/intent/api/v1/issues')
    get_dnac_site_health('dna/intent/api/v1/site-health')
    get_dnac_network_device('api/v1/network-device')
    get_dnac_interfaces('api/v1/interface')
    org_resp, api_status = meraki_http_get(meraki_url='organizations')
    meraki_orgs_to_db(org_resp)
    meraki_devices_to_db()
    meraki_networks_to_db()
    aci_cpu_status_update()
    aci_fabric_node_update()
    aci_fvBD_update()
    aci_fvAEPg_update()
    aci_firmwareRunning_update()
    fabricNodeHealth()
    aci_eqptlc_update()
    aci_eqptingrtotal1d_update()
    aci_bgpPeerEntry_update()
    aci_fault_update()
    aci_ethpmphysif_update()

```

> Note - Consider adding a **time.sleep(2)** function or similar so you don't DDoS your APIC :)

Run the cisco_grafana_main.py app:
```shell
(venv) % python cisco_grafana_main.py
API Request Successful for fabricNode.json
API Request Successful for fvBD.json?rsp-subtree=full
API Request Successful for fvAEPg.json?rsp-subtree=full
API Request Successful for firmwareRunning.json?query-target=self
API Request Successful for fabricNodeHealth.json?query-target=self
API Request Successful for eqptLC.json?query-target=self
API Request Successful for eqptIngrTotal1d.json?query-target=self
API Request Successful for procSysCPU5min.json?query-target=self
```

---

# 7.4.7 Django Tables2

The [Django Tables2](https://django-tables2.readthedocs.io/en/latest/pages/installation.html) Module provides a web frontend that allows the viewing of our Postgres (or other database) tables.

Install django tables2: `pip install django-tables2`

And ensure django_tables2 is added to settings.py > Installed_Apps
> After installing, add 'django_tables2' to INSTALLED_APPS and make sure that "django.template.context_processors.request" is added to the context_processors in your template setting OPTIONS.

Create the following in aci > tables.py:
```python
import django_tables2 as tables
from .models import aci_fabric_node

class aci_fabric_nodeTable(tables.Table):
    class Meta:
        model = aci_fabric_node
        template_name = "django_tables2/bootstrap.html"
```

aci > views.py
```python
# aci/views.py
from django_tables2 import SingleTableView

from .models import aci_fabric_node
from .tables import aci_fabric_nodeTable


class FabricNodeView(SingleTableView):
    model = aci_fabric_node
    table_class = aci_fabric_nodeTable
    template_name = 'aci/fabric-node.html'
```

In aci > templates > aci > fabric-node.html:
![Fabric Node Template](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/fabric-node-template.png?raw=true)
> Sorry, this web-site's trying to interpret/process the jinja2 tags, you'll have to make do with an image.  Refer to the django-tables2 [tutorial](https://django-tables2.readthedocs.io/en/latest/pages/tutorial.html) if you prefer to copy and paste.

And in the base cisco-grafana urls.py file:

```python
from django.contrib import admin
from django.urls import path

from aci.views import FabricNodeView

urlpatterns = [
    path('admin/', admin.site.urls),
    path("aci-fabric-node/", FabricNodeView.as_view()),
]
```

Start django server `python manage.py runserver 0:8080` and browse to http://localhost:8080/aci-fabric-node/

![Fabric Node Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/fabric_node_table.png?raw=true)

Repeat the same for remaining models / Postgres tables that we've created.  Reminder on the steps:
* Define the table under `aci > table.py`
* Create the view and `aci > views.py`
* Create the HTML template under `aci > templates > aci > <table-name>.html`
* Add the URL pattern to `urls.py`

Some examples....

##### EPGS
![EPG Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/EPG_Table.png?raw=true)

##### BDs
![BD Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/BD_table.png?raw=true)

##### Faults
![Faults Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Fault_Table.png?raw=true)

##### Firmware
![Firmware Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/FW_Table.png?raw=true)

##### BGP
![BGP Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/BGP_Table.png?raw=true)

The same process can be used for the tables we built for the previous architectures we've covered, to create a window into the database tables holding all the information we've captured across environments.  Using markdown language, we can create hyperlinks to these tables within a textbox in our grafana dashboards.

# 7.4.8 Grafana

Let's create our dashboard to show, device and interface detail, issues, software versions, network health and text panels using markdown language.

* Create a new dashboard
* Add a new panel to show CPU Utilisation over time
* Time Column = **last_updated**
* Metric Column = **dn**
* From the select column select **user_avg** & **kern_avg** & **idle_avg**
* Select **Time Series** from the right
* Configure the table as needed

Your preview pane should now look like below - click save and apply
![Network Health](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/ACI_CPU_graph.png?raw=true)

* Add another panel
* Table = **aci_aci_cpu_status**
* Time Column = **last_updated**
* Metric Column = **version**
* From the select column select **column : version** & **aggregate : count** & **alias : version**
* Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Repeat as needed for interfaces, software or other metrics your interested in - see the dashboard example at the top of this page for examples.

Your preview pane should now look like below - click save and apply
![Platform Overview](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/ACI_Version_pie.png?raw=true)

Create a new panel to hold device detail as below
* From **aci_aci_fabric_node**
* Time Column = **last_updated**
* Metric Column = None
* Format as table
* Add required columns

Your preview pane should now look like below - click save and apply.
![Devices Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/ACI_Node_table.png?raw=true)

Add other panels to your dashboard to suit your requirements.  There's some examples in the dashboard shown at the top of this page.

# 7.5 Aggregated dashboards

Some examples of creating aggregated dashboards by function across technology domains below.  

##### Firmware
![Firmware](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/aggregate_firmware.png?raw=true)

##### Performance and Network Health
![Performance Health](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/aggregate_perf_health.png?raw=true)
