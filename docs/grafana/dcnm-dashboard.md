---
layout: default
title: 3.2 Building a DCNM Dashboard
parent: Grafana Django and Cisco APIs Dashboard
nav_order: 5
last_modified_date: 2021-05-05 08:00:34 +1200
---
# Part (3.2) DCNM and VXLAN BGP and EVPN Lab with Nexus 9000v

## 3.2.1 Overview
If you've completed section [3.1 DCNM VXLAN BGP and EVPN Lab](https://j-sulliman.github.io/2021/05/04/Part-3.1-DCNM-Lab-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html), you'll now have a DCNM and N9000v EVPN and VXLAN Virtual Sandbox to test our APIs and visualisations with Python, Django and Grafana.

We stepped through an overview of a Grafana Dashboard last week, this week, we'll be focussing on DCNM as shown in green:
![Solution Overview](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Grafana_overview_DCNM.png?raw=true)

By the end of this article we'll have:
 * Gained familiarity with DCNM APIs and python requests
 * Experimented with API endpoints outlined in the API reference and the output of queries
 * Defined our Django Model classes and written output to our PostgresSQL database
 * Experimented with visualisations of the database with grafana

## 3.2.2 Resources
* [DCNM REST API Reference Guide](https://developer.cisco.com/docs/data-center-network-manager/11-5-1/)
* [Easy Provisioning of VXLAN BGP EVPN Fabrics](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/managing-greenfield-vxlan-fabric.html)
* [Auto-Provisioning Border Gateways with Multi-Site Domains](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/border-provisioning-multisite.html )
* [VXLAN EVPN Multi-Site Design and Deployment Whitepaper](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-9000-series-switches/white-paper-c11-739942.html)
* [Cisco Communities - Data Center](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/border-provisioning-multisite.html )

## 3.2.3 Django Setup

### 3.2.3.1 Create the DCNM App (Directory Structure) in django
```python
$ python manage.py startapp dcnm
$ ls dcnm/
admin.py  apps.py  __init__.py  migrations  models.py  tests.py  views.py
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
    'meraki',
    'dcnm', # Add this
]
```
## 3.2.4 Python DCNM Requests functions

### 3.2.4.1 DCNM API Requests functions
The *dcnm_http_get()* function in turn calls *get_auth_token()* which:
* Sends an HTTP Post Request to *https://[dcnm-mgmt]/rest/logon* with a user name and password
* Takes the token from the *Dcnm-Token* key in the JSON response
* Uses this token to authenticate API calls to other API endpoints, *https://[DCNM-MGMT]/rest/inventory/switches*, in our example
* "Pretty Prints" the output *pp.pprint(device_list)* for validation

```python

DCNM_USER = "admin"
DCNM_PASSWORD = "Cisco123!"
dcnm = '192.168.200.230'

def get_auth_token():
    """
    Building out Auth request. Using requests.post to make a call to the Auth Endpoint
    """
    url = 'https://{}/rest/logon'.format(dcnm)       # Endpoint URL
    data= '{"expirationTime": 360000}'
    resp = requests.post(url, auth=HTTPBasicAuth(DCNM_USER, DCNM_PASSWORD), verify=False, data=data)  # Make the POST Request
    token = resp.json()['Dcnm-Token']    # Retrieve the Token from the returned JSON
    print("Token Retrieved: {}".format(token))  # Print out the Token
    return token    # Create a return statement to send the token back for later use


def dcnm_http_get(dcnm_class):
    token = get_auth_token() # Get a Token
    url = url = 'https://{}/rest{}'.format(dcnm, dcnm_class)  #Network Device endpoint
    payload = {}
    hdr = {'Dcnm-Token': token, 'content-type' : 'application/json'}
    querystring = ''
    resp = requests.get(url, headers=hdr, verify=False, data=payload)  # Make the Get Request
    device_list = resp.json() # Capture data from the controller
    pp.pprint(device_list)
    return device_list # Pretty print the data

```

Run the functions and validate the output - we'll start with the */inventory/switches* endpoint in our example.

```python
dcnm_http_get('/inventory/switches')

[{'activeSupSlot': 0,
  'availPorts': 0,
  'colDBId': 0,
  'connUnitStatus': 0,
  'consistencyState': False,
  'contact': '',
  'cpuUsage': 39,
  'displayHdrs': ['Name',
                  'IP Address',
                  'Fabric',
                  'WWN',
                  'FC Ports',
                  'Vendor',
                  'Model',
                  'Release',
                  'UpTime'],
  'displayValues': ['dc1-bdlf-001',
                    '192.168.200.228',
                    'akl-fabric-01',
                    '9ENNZ72SXGG',
                    '64',
                    'Cisco',
                    'N9K-C9300v',
                    '9.3(7)',
                    '1 day, 11:12:03'],
  'domain': None,
  'domainID': 0,
  'fabricName': 'akl-fabric-01',
  'fcoeEnabled': False,
  'fex': False,
  'fexMap': {},
  'fid': 3,
  'health': 54,
  'hostName': 'dc1-bdlf-001',
  'index': 1,
  'ipAddress': '192.168.200.228',
  'ipDomain': '',
  'isLan': False,
  'isNonNexus': False,
  'isPmCollect': False,
  'isVpcConfigured': False,
  'is_smlic_enabled': False,
  'keepAliveState': None,
  'lastScanTime': 1620247841164,
  'licenseDetail': None,
  'licenseViolation': False,
  'linkName': None,
  'location': '',
  'logicalName': 'dc1-bdlf-001',
  'managable': True,
  'mds': False,
  'membership': None,
  'memoryUsage': 51,
  'mgmtAddress': None,
  'mode': 'Normal',
  'model': 'N9K-C9300v',
  'modelType': 0,
  'modules': None,
  'name': None,
  'network': 'LAN',
  'nonMdsModel': 'N9K-C9300v',
  'npvEnabled': False,
  'numberOfPorts': 64,
  'peer': None,
  'peerSerialNumber': None,
  'peerSwitchDbId': 0,
  'peerlinkState': None,
  'ports': 0,
  'present': True,
  'primaryIP': None,
  'primarySwitchDbID': 0,
  'principal': None,
  'recvIntf': None,
  'release': '9.3(7)',
  'role': None,
  'sanAnalyticsCapable': False,
  'scope': None,
  'secondaryIP': '',
  'secondarySwitchDbID': 0,
  'sendIntf': None,
  'serialNumber': '9ENNZ72SXGG',
  'standbySupState': 0,
  'status': 'ok',
  'swWwn': None,
  'swWwnName': '9ENNZ72SXGG',
  'switchDbID': 127070,
  'switchRole': 'border gateway',
  'uid': 0,
  'unmanagableCause': '',
  'upTime': 12672314,
  'upTimeNumber': 0,
  'upTimeStr': '1 day, 11:12:03',
  'usedPorts': 0,
  'username': 'V+g7SgiCGkvf3Q8BS91XoLZZxPWGVQkBGkWei8oIwy1e',
  'vdcId': -1,
  'vdcMac': None,
  'vdcName': '',
  'vendor': 'Cisco',
  'version': None,
  'vpcDomain': 0,
  'vsanWwn': None,
  'vsanWwnName': None,
  'wwn': None},
```

Re-run the function this time using the endpoint / dn -  */control/fabrics* and validate the output

```python
dcnm_http_get('/control/fabrics')
```
You should receive output as below
```python
[{'asn': '65001',
  'createdOn': 1619997808399,
  'deviceType': 'n9k',
  'fabricId': 'FABRIC-3',
  'fabricName': 'akl-fabric-01',
  'fabricTechnology': 'VXLANFabric',
  'fabricTechnologyFriendly': 'VXLAN Fabric',
  'fabricType': 'Switch_Fabric',
  'fabricTypeFriendly': 'Switch Fabric',
  'id': 3,
  'modifiedOn': 1620125447004,
  'networkExtensionTemplate': 'Default_Network_Extension_Universal',
  'networkTemplate': 'Default_Network_Universal',
  'nvPairs': {'AAA_REMOTE_IP_ENABLED': 'false',
              'AAA_SERVER_CONF': '',
              'ACTIVE_MIGRATION': 'false',
              'ADVERTISE_PIP_BGP': 'false',
              'AGENT_INTF': 'eth0',
              'ANYCAST_GW_MAC': '2020.0000.00aa',
              'ANYCAST_LB_ID': '',
              'ANYCAST_RP_IP_RANGE': '10.254.254.0/24',
              'ANYCAST_RP_IP_RANGE_INTERNAL': '10.254.254.0/24',
              'AUTO_SYMMETRIC_VRF_LITE': 'false',
              'BFD_AUTH_ENABLE': 'false',
              'BFD_AUTH_KEY': '',
              'BFD_AUTH_KEY_ID': '',
              'BFD_ENABLE': 'false',
              'BFD_IBGP_ENABLE': 'false',
              'BFD_ISIS_ENABLE': 'false',
              'BFD_OSPF_ENABLE': 'false',
              'BFD_PIM_ENABLE': 'false',
              'BGP_AS': '65001',
              'BGP_AUTH_ENABLE': 'false',
              'BGP_AUTH_KEY': '',
              'BGP_AUTH_KEY_TYPE': '3',
              'BGP_LB_ID': '0',
              'BOOTSTRAP_CONF': '',
              'BOOTSTRAP_ENABLE': 'false',
              'BOOTSTRAP_MULTISUBNET': '',
              'BOOTSTRAP_MULTISUBNET_INTERNAL': '',
              'BRFIELD_DEBUG_FLAG': 'Disable',
              'BROWNFIELD_NETWORK_NAME_FORMAT': 'Auto_Net_VNI$$VNI$$_VLAN$$VLAN_ID$$',
              'CDP_ENABLE': 'false',
              'COPP_POLICY': 'strict',
              'DCI_SUBNET_RANGE': '10.33.0.0/16',
              'DCI_SUBNET_TARGET_MASK': '30',
              'DEAFULT_QUEUING_POLICY_CLOUDSCALE': 'queuing_policy_default_8q_cloudscale',
              'DEAFULT_QUEUING_POLICY_OTHER': 'queuing_policy_default_other',
              'DEAFULT_QUEUING_POLICY_R_SERIES': 'queuing_policy_default_r_series',
              'DEPLOYMENT_FREEZE': 'false',
              'DHCP_ENABLE': 'false',
              'DHCP_END': '',
              'DHCP_END_INTERNAL': '',
              'DHCP_IPV6_ENABLE': 'DHCPv4',
              'DHCP_IPV6_ENABLE_INTERNAL': '',
              'DHCP_START': '',
              'DHCP_START_INTERNAL': '',
              'DNS_SERVER_IP_LIST': '',
              'DNS_SERVER_VRF': '',
              'ENABLE_AAA': 'false',
              'ENABLE_AGENT': 'false',
              'ENABLE_DEFAULT_QUEUING_POLICY': 'false',
              'ENABLE_EVPN': 'true',
              'ENABLE_FABRIC_VPC_DOMAIN_ID': 'false',
              'ENABLE_FABRIC_VPC_DOMAIN_ID_PREV': 'false',
              'ENABLE_MACSEC': 'false',
              'ENABLE_NGOAM': 'true',
              'ENABLE_NXAPI': 'true',
              'ENABLE_NXAPI_HTTP': 'true',
              'ENABLE_PBR': 'false',
              'ENABLE_TENANT_DHCP': 'true',
              'ENABLE_TRM': 'false',
              'ENABLE_VPC_PEER_LINK_NATIVE_VLAN': 'false',
              'EXTRA_CONF_INTRA_LINKS': '',
              'EXTRA_CONF_LEAF': '',
              'EXTRA_CONF_SPINE': '',
              'FABRIC_INTERFACE_TYPE': 'p2p',
              'FABRIC_MTU': '9216',
              'FABRIC_MTU_PREV': '9216',
              'FABRIC_NAME': 'akl-fabric-01',
              'FABRIC_TYPE': 'Switch_Fabric',
              'FABRIC_VPC_DOMAIN_ID': '',
              'FABRIC_VPC_DOMAIN_ID_PREV': '',
              'FABRIC_VPC_QOS': 'false',
              'FABRIC_VPC_QOS_POLICY_NAME': 'spine_qos_for_fabric_vpc_peering',
              'FEATURE_PTP': 'false',
              'FEATURE_PTP_INTERNAL': 'false',
              'FF': 'Easy_Fabric',
              'GRFIELD_DEBUG_FLAG': 'Enable',
              'HD_TIME': '180',
              'IBGP_PEER_TEMPLATE': '',
              'IBGP_PEER_TEMPLATE_LEAF': '',
              'ISIS_AUTH_ENABLE': 'false',
              'ISIS_AUTH_KEY': '',
              'ISIS_AUTH_KEYCHAIN_KEY_ID': '',
              'ISIS_AUTH_KEYCHAIN_NAME': '',
              'ISIS_LEVEL': 'level-2',
              'ISIS_P2P_ENABLE': '',
              'L2_HOST_INTF_MTU': '9216',
              'L2_HOST_INTF_MTU_PREV': '9216',
              'L2_SEGMENT_ID_RANGE': '30000-49000',
              'L3VNI_MCAST_GROUP': '',
              'L3_PARTITION_ID_RANGE': '50000-59000',
              'LINK_STATE_ROUTING': 'ospf',
              'LINK_STATE_ROUTING_TAG': 'UNDERLAY',
              'LINK_STATE_ROUTING_TAG_PREV': 'UNDERLAY',
              'LOOPBACK0_IPV6_RANGE': '',
              'LOOPBACK0_IP_RANGE': '10.2.0.0/22',
              'LOOPBACK1_IPV6_RANGE': '',
              'LOOPBACK1_IP_RANGE': '10.3.0.0/22',
              'MACSEC_ALGORITHM': '',
              'MACSEC_CIPHER_SUITE': '',
              'MACSEC_FALLBACK_ALGORITHM': '',
              'MACSEC_FALLBACK_KEY_STRING': '',
              'MACSEC_KEY_STRING': '',
              'MACSEC_REPORT_TIMER': '',
              'MGMT_GW': '',
              'MGMT_GW_INTERNAL': '',
              'MGMT_PREFIX': '',
              'MGMT_PREFIX_INTERNAL': '',
              'MGMT_V6PREFIX': '',
              'MGMT_V6PREFIX_INTERNAL': '',
              'MPLS_HANDOFF': 'false',
              'MPLS_LB_ID': '',
              'MPLS_LOOPBACK_IP_RANGE': '',
              'MSO_CONNECTIVITY_DEPLOYED': '',
              'MSO_CONTROLER_ID': '',
              'MSO_SITE_GROUP_NAME': '',
              'MSO_SITE_ID': '',
              'MULTICAST_GROUP_SUBNET': '239.1.1.0/25',
              'NETWORK_VLAN_RANGE': '2300-2999',
              'NTP_SERVER_IP_LIST': '',
              'NTP_SERVER_VRF': '',
              'NVE_LB_ID': '1',
              'OSPF_AREA_ID': '0.0.0.0',
              'OSPF_AUTH_ENABLE': 'false',
              'OSPF_AUTH_KEY': '',
              'OSPF_AUTH_KEY_ID': '',
              'PHANTOM_RP_LB_ID1': '',
              'PHANTOM_RP_LB_ID2': '',
              'PHANTOM_RP_LB_ID3': '',
              'PHANTOM_RP_LB_ID4': '',
              'PIM_HELLO_AUTH_ENABLE': 'false',
              'PIM_HELLO_AUTH_KEY': '',
              'POWER_REDUNDANCY_MODE': 'ps-redundant',
              'PREMSO_PARENT_FABRIC': '',
              'PTP_DOMAIN_ID': '',
              'PTP_LB_ID': '',
              'REPLICATION_MODE': 'Multicast',
              'ROUTER_ID_RANGE': '',
              'ROUTE_MAP_SEQUENCE_NUMBER_RANGE': '1-65534',
              'RP_COUNT': '2',
              'RP_LB_ID': '254',
              'RP_MODE': 'asm',
              'RR_COUNT': '2',
              'SERVICE_NETWORK_VLAN_RANGE': '3000-3199',
              'SITE_ID': '65001',
              'SNMP_SERVER_HOST_TRAP': 'true',
              'SPINE_COUNT': '2',
              'SSPINE_ADD_DEL_DEBUG_FLAG': 'Disable',
              'SSPINE_COUNT': '0',
              'STATIC_UNDERLAY_IP_ALLOC': 'false',
              'STRICT_CC_MODE': 'false',
              'SUBINTERFACE_RANGE': '2-511',
              'SUBNET_RANGE': '10.4.0.0/16',
              'SUBNET_TARGET_MASK': '30',
              'SYSLOG_SERVER_IP_LIST': '',
              'SYSLOG_SERVER_VRF': '',
              'SYSLOG_SEV': '',
              'TCAM_ALLOCATION': 'true',
              'UNDERLAY_IS_V6': 'false',
              'USE_LINK_LOCAL': 'true',
              'V6_SUBNET_RANGE': '',
              'V6_SUBNET_TARGET_MASK': '126',
              'VPC_AUTO_RECOVERY_TIME': '360',
              'VPC_DELAY_RESTORE': '150',
              'VPC_DELAY_RESTORE_TIME': '60',
              'VPC_DOMAIN_ID_RANGE': '1-1000',
              'VPC_ENABLE_IPv6_ND_SYNC': 'true',
              'VPC_PEER_KEEP_ALIVE_OPTION': 'management',
              'VPC_PEER_LINK_PO': '500',
              'VPC_PEER_LINK_VLAN': '3600',
              'VRF_LITE_AUTOCONFIG': 'Manual',
              'VRF_VLAN_RANGE': '2000-2299',
              'abstract_anycast_rp': 'anycast_rp',
              'abstract_bgp': 'base_bgp',
              'abstract_bgp_neighbor': 'evpn_bgp_rr_neighbor',
              'abstract_bgp_rr': 'evpn_bgp_rr',
              'abstract_dhcp': 'base_dhcp',
              'abstract_extra_config_bootstrap': 'extra_config_bootstrap_11_1',
              'abstract_extra_config_leaf': 'extra_config_leaf',
              'abstract_extra_config_spine': 'extra_config_spine',
              'abstract_feature_leaf': 'base_feature_leaf_upg',
              'abstract_feature_spine': 'base_feature_spine_upg',
              'abstract_isis': 'base_isis_level2',
              'abstract_isis_interface': 'isis_interface',
              'abstract_loopback_interface': 'int_fabric_loopback_11_1',
              'abstract_multicast': 'base_multicast_11_1',
              'abstract_ospf': 'base_ospf',
              'abstract_ospf_interface': 'ospf_interface_11_1',
              'abstract_pim_interface': 'pim_interface',
              'abstract_route_map': 'route_map',
              'abstract_routed_host': 'int_routed_host_11_1',
              'abstract_trunk_host': 'int_trunk_host_11_1',
              'abstract_vlan_interface': 'int_fabric_vlan_11_1',
              'abstract_vpc_domain': 'base_vpc_domain_11_1',
              'default_network': 'Default_Network_Universal',
              'default_vrf': 'Default_VRF_Universal',
              'enableRealTimeBackup': 'false',
              'enableScheduledBackup': 'false',
              'network_extension_template': 'Default_Network_Extension_Universal',
              'scheduledTime': '',
              'software_image': '',
              'temp_anycast_gateway': 'anycast_gateway',
              'temp_vpc_domain_mgmt': 'vpc_domain_mgmt',
              'temp_vpc_peer_link': 'int_vpc_peer_link_po_11_1',
              'vrf_extension_template': 'Default_VRF_Extension_Universal'},
  'provisionMode': 'DCNMTopDown',
  'replicationMode': 'Multicast',
  'siteId': '65001',
  'templateName': 'Easy_Fabric_11_1',
  'vrfExtensionTemplate': 'Default_VRF_Extension_Universal',
  'vrfTemplate': 'Default_VRF_Universal'},
 {'createdOn': 1620121501616,
```
We've retrieved a lot of information from the above API calls and dictionary keys, including software versions, models, hostnames, uptime, BGP ASN, Fabric IP, VRF VLAN ranges that we can begin working with.  

## 3.2.5  Django Model (Database Table) Definition

Create our models for DCNM devices in *dcnm/models.py* and define columns/entries for the dictionary keys of interest:

```python
# Create your models here.
class dcnm_network_device(models.Model):
    dcnm_addr = models.CharField(max_length=200)
    asn = models.CharField(max_length=200)
    deviceType = models.CharField(max_length=200)
    fabricId = models.CharField(max_length=200)
    fabricName = models.CharField(primary_key=True, max_length=200)
    fabricTechnology = models.CharField(max_length=200)
    fabricType = models.CharField(max_length=200)
    id = models.CharField(max_length=200)
    networkTemplate = models.CharField(max_length=200)
    ADVERTISE_PIP_BGP = models.CharField(max_length=200)
    ANYCAST_GW_MAC = models.CharField(max_length=200)
    ANYCAST_RP_IP_RANGE = models.CharField(max_length=200)
    BFD_ENABLE = models.CharField(max_length=200)
    BFD_AUTH_ENABLE = models.CharField(max_length=200)
    BGP_AS = models.CharField(max_length=200)
    BGP_AUTH_ENABLE = models.CharField(max_length=200)
    DCI_SUBNET_RANGE = models.CharField(max_length=200)
    DCI_SUBNET_TARGET_MASK = models.CharField(max_length=200)
    DHCP_ENABLE = models.CharField(max_length=200)
    ENABLE_EVPN = models.CharField(max_length=200)
    ENABLE_NXAPI = models.CharField(max_length=200)
    FABRIC_INTERFACE_TYPE = models.CharField(max_length=200)
    FABRIC_MTU = models.CharField(max_length=200)
    FABRIC_NAME = models.CharField(max_length=200)
    L2_SEGMENT_ID_RANGE = models.CharField(max_length=200)
    L3_PARTITION_ID_RANGE = models.CharField(max_length=200)
    LINK_STATE_ROUTING = models.CharField(max_length=200)
    LINK_STATE_ROUTING_TAG = models.CharField(max_length=200)
    LOOPBACK0_IP_RANGE = models.CharField(max_length=200)
    LOOPBACK1_IP_RANGE = models.CharField(max_length=200)
    MULTICAST_GROUP_SUBNET = models.CharField(max_length=200)
    NETWORK_VLAN_RANGE = models.CharField(max_length=200)
    OSPF_AREA_ID = models.CharField(max_length=200)
    OSPF_AUTH_ENABLE = models.CharField(max_length=200)
    REPLICATION_MODE = models.CharField(max_length=200)
    RR_COUNT = models.CharField(max_length=200)
    SERVICE_NETWORK_VLAN_RANGE = models.CharField(max_length=200)
    SUBNET_RANGE = models.CharField(max_length=200)
    replicationMode = models.CharField(max_length=200)
    templateName = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.switch_name

class dcnm_network_switch(models.Model):
    dcnm_addr = models.CharField(max_length=200)
    activeSupSlot = models.CharField(max_length=200)
    availPorts = models.IntegerField()
    consistencyState = models.CharField(max_length=200)
    cpuUsage = models.IntegerField()
    fabricName = models.CharField(max_length=200)
    health = models.IntegerField()
    hostName = models.CharField(max_length=200)
    ipAddress = models.CharField(max_length=200)
    isVpcConfigured = models.CharField(max_length=200)
    keepAliveState = models.CharField(max_length=200, default='None')
    licenseDetail = models.CharField(max_length=200)
    licenseViolation = models.CharField(max_length=200)
    logicalName = models.CharField(max_length=200)
    managable = models.CharField(max_length=200)
    memoryUsage = models.CharField(max_length=200)
    mode = models.CharField(max_length=200)
    model = models.CharField(max_length=200)
    network = models.CharField(max_length=200)
    numberOfPorts = models.CharField(max_length=200)
    peer = models.CharField(max_length=200)
    present = models.CharField(max_length=200)
    release = models.CharField(max_length=200)
    serialNumber = models.CharField(max_length=200)
    upTime = models.BigIntegerField()
    vendor = models.CharField(max_length=200)
    vpcDomain = models.CharField(max_length=200)
    last_updated = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.switch_name
```
Commit the models to the database:
```python
% python manage.py makemigrations dcnm
Migrations for 'dcnm':
  dcnm/migrations/0001_initial.py
    - Create model dcnm_network_device
    - Create model dcnm_network_switch
% python manage.py migrate dcnm       
Operations to perform:
  Apply all migrations: dcnm
```

### 3.2.5.1 Writing output to Database with Djangos Database API
When satisfied with above API calls:
* Import our new dcnm_network_device and dcnm_network_switch models into dcnm_api.py
* Iterate over each item in our list of output
* Map the keys of interest in the JSON dictionary output to our model's fields
* Commit the entry to the database with *device_entry.save()*

```python

from dcnm.models import dcnm_network_device, dcnm_network_switch
def get_dcnm_network_device(query_class):
            try:
                devices = dcnm_http_get(query_class)
                for device in devices:
                    if device['fabricType'] == 'Switch_Fabric':
                        device_entry=dcnm_network_device(
                            dcnm_addr = dcnm,
                            asn = device['asn'],
                            deviceType = device['deviceType'],
                            fabricId = device['fabricId'],
                            fabricName = device['fabricName'],
                            fabricTechnology = device['fabricTechnology'],
                            fabricType = device['fabricType'],
                            id = device['id'],
                            networkTemplate = device['networkTemplate'],
                            ADVERTISE_PIP_BGP = device['nvPairs']['ADVERTISE_PIP_BGP'],
                            ANYCAST_GW_MAC = device['nvPairs']['ANYCAST_GW_MAC'],
                            ANYCAST_RP_IP_RANGE = device['nvPairs']['ANYCAST_RP_IP_RANGE'],
                            BFD_ENABLE = device['nvPairs']['BFD_ENABLE'],
                            BFD_AUTH_ENABLE = device['nvPairs']['BFD_AUTH_ENABLE'],
                            BGP_AS = device['nvPairs']['BGP_AS'],
                            BGP_AUTH_ENABLE = device['nvPairs']['BGP_AUTH_ENABLE'],
                            DCI_SUBNET_RANGE = device['nvPairs']['DCI_SUBNET_RANGE'],
                            DCI_SUBNET_TARGET_MASK = device['nvPairs']['DCI_SUBNET_TARGET_MASK'],
                            DHCP_ENABLE = device['nvPairs']['DHCP_ENABLE'],
                            ENABLE_EVPN = device['nvPairs']['ENABLE_EVPN'],
                            ENABLE_NXAPI = device['nvPairs']['ENABLE_NXAPI'],
                            FABRIC_INTERFACE_TYPE = device['nvPairs']['FABRIC_INTERFACE_TYPE'],
                            FABRIC_MTU = device['nvPairs']['FABRIC_MTU'],
                            FABRIC_NAME = device['nvPairs']['FABRIC_NAME'],
                            L2_SEGMENT_ID_RANGE = device['nvPairs']['L2_SEGMENT_ID_RANGE'],
                            L3_PARTITION_ID_RANGE = device['nvPairs']['L3_PARTITION_ID_RANGE'],
                            LINK_STATE_ROUTING = device['nvPairs']['LINK_STATE_ROUTING'],
                            LINK_STATE_ROUTING_TAG = device['nvPairs']['LINK_STATE_ROUTING_TAG'],
                            LOOPBACK0_IP_RANGE = device['nvPairs']['LOOPBACK0_IP_RANGE'],
                            LOOPBACK1_IP_RANGE = device['nvPairs']['LOOPBACK1_IP_RANGE'],
                            MULTICAST_GROUP_SUBNET = device['nvPairs']['MULTICAST_GROUP_SUBNET'],
                            NETWORK_VLAN_RANGE = device['nvPairs']['NETWORK_VLAN_RANGE'],
                            OSPF_AREA_ID = device['nvPairs']['OSPF_AREA_ID'],
                            OSPF_AUTH_ENABLE = device['nvPairs']['OSPF_AUTH_ENABLE'],
                            REPLICATION_MODE = device['nvPairs']['REPLICATION_MODE'],
                            RR_COUNT = device['nvPairs']['RR_COUNT'],
                            SERVICE_NETWORK_VLAN_RANGE = device['nvPairs']['SERVICE_NETWORK_VLAN_RANGE'],
                            SUBNET_RANGE = device['nvPairs']['SUBNET_RANGE'],
                            replicationMode = device['replicationMode'],
                            templateName = device['templateName'])
                        device_entry.save()
                aci_model_pruner(dnac_network_device.objects.all(), obj_type="DCNM Fabric", retention_period=30)
            except:
                print('Unable to query devices on DCNM {}'.format(dcnm))


def get_dcnm_network_switch(query_class):
    try:
        devices = dcnm_http_get(query_class)
        for device in devices:
            device_entry=dcnm_network_switch(
                dcnm_addr = dcnm,
                activeSupSlot = device['activeSupSlot'],
                availPorts = device['availPorts'],
                consistencyState = device['consistencyState'],
                cpuUsage = device['cpuUsage'],
                fabricName = device['fabricName'],
                health = device['health'],
                hostName = device['hostName'],
                ipAddress = device['ipAddress'],
                isVpcConfigured = device['isVpcConfigured'],
                licenseViolation = device['licenseViolation'],
                logicalName = device['logicalName'],
                managable = device['managable'],
                memoryUsage = device['memoryUsage'],
                mode = device['mode'],
                model = device['model'],
                network = device['network'],
                numberOfPorts = device['numberOfPorts'],
                present = device['present'],
                release = device['release'],
                serialNumber = device['serialNumber'],
                upTime = device['upTime'],
                vendor = device['vendor'],
                vpcDomain = device['vpcDomain']
                )
            device_entry.save()
        aci_model_pruner(dnac_network_device.objects.all(), obj_type="DCNM Switch", retention_period=30)
    except:
        print('Unable to query devices on DCNM {}'.format(dcnm))
```

Run the functions:
```python
get_dcnm_network_switch('/inventory/switches')
get_dcnm_network_device('/control/fabrics')
```

As we did in the meraki tutorial, add the following to dcnm_api.py so that it knows where to find our django settings and model files:

```python
import os
import sys

# Add your own path to django install location here
sys.path.append("/Users/jamespsullivan/Documents/grafana-cisco-demo/cisco_grafana/")

# We need to import django configuration settings to write our API response using the django models API
os.environ['DJANGO_SETTINGS_MODULE'] = 'cisco_grafana.settings'

import django
django.setup()
```
Use PgAdmin to verify devices have been retrieved and written to the database
![pgAdmin](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_PGAdmin.png?raw=true)


## 3.2.6 Grafana Visualisations

#### Installation
Refer to Tutuorial (2) Meraki for Grafana installation and setup instructions

Create a new dashboard and add a new panel - configure **query options** as below.  Note - if you don't configure __"aggregate: count"__ as shown, you'll most likely receive an error.

Select PieChartv2 from the right, optionally change the PieChart type to __donut__, add the labels name, value and percent and give the chart a title.

Your preview pane should now look like below - click save and apply
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_Software_Versions.png?raw=true)

Add a new panel to display device health information:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_Bar_Gauge.png?raw=true)

CPU and Memory utilisation can be monitored with a Gauge Panel as shown:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_CPU_Gauge.png?raw=true)

* Return to the dashboard and add another panel, again select the **dcnm_dcnm_network_switch** table.
* From the right hand pane, select **table**
* In the **query options**, select "format as: table"
* Add columns of interest to the table

If you've managed to follow this tutorial successfully, your DCNM panel should now look like below:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/DCNM_Dashboard.png?raw=true)


## 3.2.7 Wrap Up

That's it for this week's update thanks for reading through.  We've only scratched the surface of DCNM APIs and visualisations, but I hope that there's some useful content you can use to continue exploring Cisco APIs and opensource software such as grafana.

I'll update this site's as I cover remaining architectures - SDWAN and vManage up next.

Jamie
