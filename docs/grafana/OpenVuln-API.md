---
layout: default
title: 5.0 Cisco Vulnerability and PSIRTS Dashboard
parent: Creating a Dashboard with Grafana Django and Cisco APIs
nav_order: 8
last_modified_date: 2021-05-23 20:00:34 +1200
---

# 5.1 Introduction

In part five of the series on cross domain visualisations and dashboards  with Grafana, we experiment with the OpenVuln API.

> The Cisco Product Security Incident Response Team (PSIRT) openVuln API is a RESTful API that allows customers to obtain Cisco Security Vulnerability information in different machine-consumable formats. APIs are important for customers because they allow their technical staff and programmers to build tools that help them do their job more effectively (in this case, to keep up with security vulnerability information).

By the end of this article we'll have created a dashboard that provides a single, aggregated view of:
* Advisories by product family
* PSIRT severity rating and when they were released
* A detailed table with links to the PSIRT URL
* Created alerting and notifications when new PSIRTs are detected

![OpenVuln Dashboard](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/OpenVuln_Dash.png?raw=true)


# 5.2 References
*  [OpenVuln API on DevNet](https://developer.cisco.com/psirt/)
*  [API Reference Guide](https://developer.cisco.com/docs/psirt/)
*  [OpenVuln GitHub Repo](https://github.com/CiscoPSIRT/openVulnAPI)
* [Grafana Alerts](https://grafana.com/docs/grafana/latest/alerting/create-alerts/)

---

# 5.3 Generating an API Key and Secret from the API Console

Following the **Accessing the API** section of the API reference guide, register a new application and select the OpenVuln API.  Once registered, you'll have an API key and secret that can be used to authenticate your requests:
![API Console](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/OpenVuln_App_and_Key.png?raw=true)


---
# 5.4 Create our OpenVuln App in Django
```shell
(venv) % python manage.py startapp openvuln
```
Running the above will create the openvuln directory structure.

If you've followed along with previous articles, your cisco_grafana django directory will look similar to:
```shell
.
├── apps
│   ├── dcnm_api.py
│   ├── meraki_api.py
│   └── sdwan_api.py
├── cisco_grafana
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── db.sqlite3
├── dcnm
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
├── meraki
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
│   │   └── __init__.py
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

Add the openvuln app to **settings.py**
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
    'openvuln', # <- Add this
]
```
---
# 5.5 Python Requests Functions

In the GitHub repository linked above, you'll find code samples for curl, go, php, ruby, javascript.

There's a python library available that can be installed via pip (`pip install openVulnQuery`).

In my case I've used the native python requests package.

Create the  `Apps > openvuln_api.py` file - this will store all our OpenVuln python functions.

Retrieving the access token:
```python
def get_oauth_token(client_id, client_secret, request_token_url="https://cloudsso.cisco.com/as/token.oauth2"):
    """Get OAuth2 token from api based on client id and secret.
    :param client_id: Client id stored in config file.
    :param client_secret: Client secret stored in config file.
    :param request_token_url: the POST URL to request a token response
    :return The valid access token to pass to api in header.
    :raise requests exhibits anything other than a 200 response.
    """

    r = requests.post(
        request_token_url if request_token_url else REQUEST_TOKEN_URL,
        params={'client_id': client_id, 'client_secret': client_secret},
        data={'grant_type': 'client_credentials'}
    )
    return r.json()['access_token']

token = get_oauth_token(client_id='<your-id>',
                        client_secret='<your-secret>')
print(token)
```

Run the function and pass in your client ID and Secret:
```shell
(venv) % python openvuln_api.py
ENPFzXc7ja9IEmAF5GCLH9mnQO5z
```

Retrieve the latest 100 vulnerabilities:
```python
def openvuln_http_get(url='https://api.cisco.com/security/advisories/latest/100'):
    CLIENT_ID = "<your-id>"
    CLIENT_SECRET = "<your-token>"
    API_URL = "https://api.cisco.com/security/advisories"
    token = get_oauth_token(client_id=CLIENT_ID, client_secret=CLIENT_SECRET) # Get a Token
    header = {
        'Authorization': 'Bearer {}'.format(token),
        'Accept': 'application/json',
        'User-Agent': 'openvuln_api',
    }
    url = url #Network Device endpoint
    querystring = ''
    resp = requests.get(url, headers=header, verify=False)  # Make the Get Request
    advisories = resp.json() # Capture data from the controller

    return advisories, resp.status_code

psirts, status = openvuln_http_get()
pp.pprint(psirts)
```

Run the function and check the output:
```shell
(venv) % python openvuln_api.py

{'advisories': [{'advisoryId': 'cisco-sa-http-fp-bp-KfDdcQhc',
                 'advisoryTitle': 'Multiple Cisco Products Snort HTTP '
                                  'Detection Engine File Policy Bypass '
                                  'Vulnerabilities',
                 'bugIDs': ['CSCvv70864',
                            'CSCvw19272',
                            'CSCvw26645',
                            'CSCvw59055'],
                 'cves': ['CVE-2021-1494', 'CVE-2021-1495'],
                 'cvrfUrl': 'https://tools.cisco.com/security/center/contentxml/CiscoSecurityAdvisory/cisco-sa-http-fp-bp-KfDdcQhc/cvrf/cisco-sa-http-fp-bp-KfDdcQhc_cvrf.xml',
                 'cvssBaseScore': '5.8',
                 'cwe': ['CWE-693'],
                 'firstPublished': '2021-04-28T16:00:00',
                 'ipsSignatures': ['NA'],
                 'lastUpdated': '2021-05-20T18:51:31',
                 'productNames': ['Cisco Firepower Threat Defense Software ',
                                  'Cisco UTD SNORT IPS Engine Software ',
                                  ],
                 'publicationUrl': 'https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-http-fp-bp-KfDdcQhc',
                 'sir': 'Medium',
                 'status': 'Final',
                 'summary': '<p><span '
                            'id="ctl00_MainBodyContainer_DgFields_ctl03_lblField">Multiple '
                            'Cisco&nbsp;products are affected by '
                            'vulnerabilities in the Snort detection engine '
                            'that</span> could allow an unauthenticated, '
                            'remote attacker to bypass a configured file '
                            'policy for HTTP.</p>\r\n'
                            '<p>These vulnerabilities are due to incorrect '
                            'handling of specific HTTP header parameters. An '
                            'attacker could exploit these vulnerabilities by '
                            'sending crafted HTTP packets through an affected '
                            'device. A successful exploit could allow the '
                            'attacker to bypass a configured file policy for '
                            'HTTP packets and deliver a malicious '
                            'payload.</p>\r\n'
                            '<p>Cisco&nbsp;has released software updates that '
                            'address these vulnerabilities. There are no '
                            'workarounds that address these '
                            'vulnerabilities.&nbsp;</p>\r\n'
                            '<p>This advisory is available at the following '
                            'link:<br><a '
                            'href="https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-http-fp-bp-KfDdcQhc" '
                            'rel="nofollow">https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-http-fp-bp-KfDdcQhc</a></p>\r\n',
                 'version': '1.1'},
```

---

# 5.6 Django Models file and Database Definition
Edit our **openvuln > models.py** file / database definition and create a model to store advisories:

```python
from django.db import models
import datetime
from django.utils import timezone

# Create your models here.
class openvuln_advisory(models.Model):
    advisoryId = models.CharField(primary_key=True, max_length=200)
    advisoryTitle = models.TextField(max_length=200)
    bugIDs = models.TextField()
    cves = models.TextField()
    cvrfUrl = models.TextField()
    cvssBaseScore = models.FloatField()
    cwe = models.CharField(max_length=200)
    firstPublished = models.DateTimeField(max_length=200)
    ipsSignatures = models.CharField(max_length=200)
    lastUpdated = models.DateTimeField(max_length=200)
    productNames = models.TextField()
    publicationUrl = models.TextField()
    sir = models.TextField()
    status = models.CharField(max_length=200)
    summary = models.TextField()
    affected_devices = models.CharField(max_length=200, default="no")
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.advisoryId
```
Commit the model to postgres sql:

```shell
(venv) % python manage.py makemigrations openvuln

Migrations for 'openvuln':
  openvuln/migrations/0001_initial.py
    - Create model openvuln_advisory
(venv) % python manage.py migrate openvuln

Operations to perform:
  Apply all migrations: openvuln
```

# 5.7 Writing to Database
Create the below function to write our advisory request output to database:
```python
# django imports
# Add your own path to django install location here
import sys
sys.path.append("/Users/jamespsullivan/Documents/grafana-cisco-demo/cisco_grafana/")
os.environ['DJANGO_SETTINGS_MODULE'] = 'cisco_grafana.settings'

import django
import json
import pprint as pp
django.setup()

# django openvuln model import
from openvuln.models import openvuln_advisory

# function to map JSON form API to DB model file and save
def advisories_to_db(advisories):
    for advisory in advisories['advisories']:
        # We want number values in this table column, convert NA to 0 value
        if advisory['cvssBaseScore'] == 'NA':
            advisory['cvssBaseScore'] = '0'
        advisory_update = openvuln_advisory(
            advisoryId = advisory['advisoryId'],
            advisoryTitle = advisory['advisoryTitle'],
            bugIDs = advisory['bugIDs'],
            cves = advisory['cves'],
            cvrfUrl = advisory['cvrfUrl'],
            cvssBaseScore = float(advisory['cvssBaseScore']),
            cwe = advisory['cwe'],
            firstPublished = advisory['firstPublished'],
            ipsSignatures = advisory['ipsSignatures'],
            lastUpdated = advisory['lastUpdated'],
            productNames = advisory['productNames'],
            publicationUrl = advisory['publicationUrl'],
            sir = advisory['sir'],
            status = advisory['status'],
            summary = advisory['summary']
        )
        advisory_update.save()

advisories_to_db(psirts)
```
Verify that the advisories have been saved to database - you can use something like below, or pgAdmin as in previous tutorials.
```python
for psirt in openvuln_advisory.objects.all():
    print(psirt.advisoryId + ', ' + psirt.advisoryTitle)
```

Output looks ok?
```shell
cisco-sa-sdwan-vmanageinfdis-LKrFpbv, Cisco SD-WAN vManage Information Disclosure Vulnerability
cisco-sa-snort-tfo-bypass-MmzZrtes, Multiple Cisco Products Snort TCP Fast Open File Policy Bypass Vulnerability
cisco-sa-20190515-nxos-cli-bypass, Cisco NX-OS Software CLI Bypass to Internal Service Vulnerability
cisco-sa-sb-wap-inject-Mp9FSdG, Cisco Small Business 100, 300, and 500 Series Wireless Access Points Command Injection Vulnerabilities
cisco-sa-finesse-strd-xss-bUKqffFW, Cisco Finesse Cross-Site Scripting Vulnerabilities
cisco-sa-dnasp-conn-cmdinj-HOj4YV5n, Cisco DNA Spaces Connector Command Injection Vulnerabilities
cisco-sa-dnasp-conn-prvesc-q6T6BzW, Cisco DNA Spaces Connector Privilege Escalation Vulnerabilities
cisco-sa-cml-cmd-inject-N4VYeQXB, Cisco Modeling Labs Web UI Command Injection Vulnerability
cisco-sa-hyperflex-rce-TjjNrkpR, Cisco HyperFlex HX Command Injection Vulnerabilities
cisco-sa-ucm-dos-OO4SRYEf, Cisco Hosted Collaboration Mediation Fulfillment Denial of Service Vulnerability
cisco-sa-anyconnect-mac-priv-esc-VqST2nrT, MacOS Local Privilege Escalation Exploitable through Cisco AnyConnect Secure Mobility Client
cisco-sa-openssl-2021-GHY28dJd, Multiple Vulnerabilities in OpenSSL Affecting Cisco Products: March 2021
cisco-sa-waas-infdisc-Twb4EypK, Cisco Wide Area Application Services Software Information Disclosure Vulnerability
cisco-sa-vmanage-xss-eN75jxtW, Cisco SD-WAN vManage API Stored Cross-Site Scripting Vulnerability
cisco-sa-wsa-xss-mVjOWchB, Cisco Web Security Appliance Cross-Site Scripting Vulnerability
cisco-sa-ise-priv-esc-fNZX8hHj, Cisco Identity Services Engine Privilege Escalation Vulnerability
cisco-sa-ftd-ssl-decrypt-dos-DdyLuK6c, Cisco Firepower Threat Defense Software SSL Decryption Policy Denial of Service Vulnerability
cisco-sa-sdwan-buffover-MWGucjtO, Cisco SD-WAN vEdge Software Buffer Overflow Vulnerabilities
cisco-sa-sdwan-arbfile-7Qhd9mCn, Cisco SD-WAN Software Arbitrary File Corruption Vulnerability
cisco-sa-amp-imm-dll-tu79hvkO, Cisco Advanced Malware Protection for Endpoints Windows Connector,  ClamAV for Windows, and Immunet DLL Hijacking Vulnerability
cisco-sa-sdwan-dos-Ckn5cVqW, Cisco SD-WAN Software vDaemon Denial of Service Vulnerability
cisco-sa-sb-wap-multi-ZAfKGXhF, Cisco Small Business 100, 300, and 500 Series Wireless Access Points Vulnerabilities
cisco-sa-nfvis-cmdinj-DkFjqg2j, Cisco Enterprise NFV Infrastructure Software Command Injection Vulnerability
cisco-sa-sd-wan-vmanage-4TbynnhZ, Cisco SD-WAN vManage Software Vulnerabilities
```
Let's add the following so we can categorise advisories by product family.

> Note:   Initial intention here was to compare the software versions collected in our previous DCNM and SD-WAN articles to the **Affected Releases** for each advisory.  Unfortunately, the API doesn't seem to currently have a field that summarises impacted software versions.  Some of the "Product Names" column reference versions, but not all... Maybe a feature request to include this in the future?

```python
for psirt in openvuln_advisory.objects.all():
    if 'NX-OS' in psirt.productNames:
        psirt.devicesImpacted = 'NX-OS'
        psirt.save()
    elif 'SD-WAN' in psirt.productNames:
        psirt.devicesImpacted = 'SD-WAN'
        psirt.save()
    elif 'IOS' in psirt.productNames:
        psirt.devicesImpacted = 'IOS XE/IOS'
        psirt.save()
    elif 'Firepower' in psirt.productNames:
        psirt.devicesImpacted = 'ASA/Firepower'
        psirt.save()
    elif 'Unified Communications' in psirt.productNames:
        psirt.devicesImpacted = 'Collab'
        psirt.save()
    elif 'Jabber' in psirt.productNames:
        psirt.devicesImpacted = 'Collab'
        psirt.save()
    elif 'Webex' in psirt.productNames:
        psirt.devicesImpacted = 'Collab'
        psirt.save()
    elif 'HyperFlex' in psirt.productNames:
        psirt.devicesImpacted = 'Hyperflex'
        psirt.save()
    else:
        psirt.devicesImpacted = 'other'
        psirt.save()
```
---
# 5.8 Grafana

* Create a new dashboard and add a new panel
* Configure **query options** as below.  We'll query by the devicesImpacted field we created previously
* Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Your preview pane should now look like below - click save and apply
![Platforms Impacted](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/OpenVuln_Platforms_Impacted.png?raw=true)

* Create a new panel to show advisories by severity and date
* Time Column = **firstPublished**
* Metric Column = **advisoryTitle**
* From the select column select **cvssBaseScore**
* Select **Bars** from the right

Your preview pane should now look like below - click save and apply
![Advisories By Date](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Advisories_by_date.png?raw=true)

Create a new panel to hold advisories detail table as below
* Time Column = **firstPublished**
* Add all relevant columns

Your preview pane should now look like below - click save and apply.  The publication URL provides a useful hyperlink to the advisory detail on tools.cisco.com.
![Advisories Table](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Openvuln_Table.png?raw=true)

# 5.9 Alerting for new Advisories

Grafana has built in alerting capabilities:
> Grafana alerting allows you to attach rules to your dashboard panels. When you save the dashboard, Grafana extracts the alert rules into a separate alert rule storage and schedules them for evaluation.

Let's give it a try.

## 5.9.1 SMTP settings and Grafana.ini

Edit grafana.ini to reflect the SMTP settings for your email account.

Note - I'm using Homebrew on mac, grafana.ini is as below - this will likely be different depending on your OS.

`(venv) % nano /opt/homebrew/etc/grafana/grafana.ini`

```
#################################### SMTP / Emailing ##########################
[smtp]
enabled = true
host = smtp.gmail.com:587
user = <your-email>@gmail.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
password = [**********]
;cert_file =
;key_file =
;skip_verify = false
from_address = admin@grafana.localhost
from_name = Grafana
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com
# SMTP startTLS policy (defaults to 'OpportunisticStartTLS')
;startTLS_policy = NoStartTLS

[emails]
;welcome_email_on_sign_up = false
;templates_pattern = emails/*.html
```
Check the status of test email from Grafana's alerting/ email test.  In my case the test failed.  No error message is given.  But from grafana.log we see below io timeout:
From `(venv) % tail /opt/homebrew/var/log/grafana/grafana.log`
```shell
(venv) % tail /opt/homebrew/var/log/grafana/grafana.log
t=2021-05-22T20:30:45+1200 lvl=eror msg="failed to send notification" logger=alerting.notifier uid= error="Failed to send notification to email addresses: sulliman@gmail.com: dial tcp 74.125.24.109:587: i/o timeout"
```

After allowing TCP 587 outbound on firewall and resending a test email:
```shell
t=2021-05-23T12:54:28+1200 lvl=info msg="Sending alert notification to" logger=alerting.notifier.email addresses=[sulliman@gmail.com] singleEmail=false
```
Test email received -
![Test Notification](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Grafana_test_notification.png?raw=true)

Edit the alert field of one of your graph panels.  In this example, I'm checking for new alerts every 12 hours.  If an advisory is published within the last 24 hours an email alert will be sent to the email address you've entered in the notifications channel.

![Advisory Alert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Advisory_Alert.png?raw=true)

You'll now receive alert notifications when new PSIRTS are detected.
![New PSIRT Notification](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/PSIRT_Notification.png?raw=true)
