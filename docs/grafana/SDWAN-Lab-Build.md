---
layout: default
title: 4.1 Cisco SDWAN Lab Build
parent: Grafana Django and Cisco APIs Dashboard
nav_order: 6
last_modified_date: 2021-05-14 08:00:00 +1200
---
# Part (4) Cisco SDWAN Virtual Lab Deployment

We've covered creating a Meraki and DCNM dashboard using APIs and Grafana in previous updates.  In this update we continue building out our Grafana dashboard with a focus on SDWAN and vManage APIs.

We'll cover SD-WAN in two parts:
* (1) Documenting the SDWAN Sandbox Lab virtual deployment
* (2) Using vManage APIs and building upon our Grafana dashboard to aggregate our visualisations across domains.

If you have your own SD-WAN environment or would rather use the [DevNet Sandbox](https://developer.cisco.com/docs/sandbox/#!networking/networking-sandbox-highlights) you can skip this article and jump straight to update (4.2) SDWAN APIs and Grafana... (once it's finished!).


| Section | Architecture | Link | Topic
------------ | ------------ | ------------- | -------------
1.0 | Introduction | [building a multidomain dashboard with cisco apis and grafana](https://j-sulliman.github.io/2021/04/26/Part-1-Intro-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Solution Overview
2.0 | Meraki | [Building a Meraki Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/04/26/Part-2-Meraki-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising Meraki APIs with Grafana.
3.1 | DCNM & EVPN VXLAN | [DCNM VXLAN BGP and EVPN Lab with Nexus 9000v](https://j-sulliman.github.io/2021/05/04/Part-3.1-DCNM-Lab-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | DCNM, N9000v, EVPN and VXLAN Lab deployment
3.2 | DCNM & EVPN VXLAN | [Building a DCNM Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/04/Part-3.2-DCNM-Building-a-Multidomain-Dashboard-with-Cisco-APIs-and-Grafana.html) | Visualising DCNM APIs with Grafana.
4.1 | SDWAN | [Cisco SDWAN Virtual Lab Build](https://j-sulliman.github.io/2021/05/13/Part-4.1-SDWAN-Cisco-SDWAN-Lab-Build.html) | Cisco SDWAN Virtual Lab deployment
4.2 | SDWAN | [Building a SD-WAN Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/17/Part-4.2-SDWAN-Cisco-SDWAN-vManage-API-Visualisations-with-Grafana.html) | Visualising vManage APIs with Grafana
5.0 | OpenVuln API | [Building a Security Advisory Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/23/Part-5.0-OpenVuln-API-Cisco-PSIRT-and-Security-Vulnerability-Dashboard-with-Grafana.html) | Visualising OpenVuln APIs with Grafana
6.0 | DNA Center| [Building a DNA-C Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/05/25/Part-6.0-DNAC-Cisco-DNA-Center-Dashboard-with-Grafana.html) | Visualising DNA-C APIs with Grafana
7.0 | ACI and Conclusion | [Building an ACI Dashboard with Grafana, Django and Python](https://j-sulliman.github.io/2021/06/02/Part-7.0-ACI-and-Conclusion-Building-a-Multidomain-Dashboard-with-ACI-APIs-and-Grafana.html) | Visualising ACI APIs with Grafana, Django send_email and django_tables2

## Resources

There's several detailed guides and resources available that were invaluable, the first three particulary so, at least in terms of building an SDWAN Lab:
* LTRRST-2734: 4 Hours to Build Your Own SDWAN Lab [Cisco Live On Demand Library](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2020/pdf/LTRRST-2734-LG.pdf)
* Doanh Luong's excellent and detailed guide [here](https://kimdoanh89.github.io/doanhluong.me/sd-wan/SD-WAN-setup/#14-viptela-cli-modes). One of my primary sources, thanks for the great write up Doanh.
* For Plug and Play, license management [codingpackets](https://codingpackets.com/blog/cisco-sdwan-self-hosted-lab-part-1/) is another good resource.
* SD-WAN and Cloud Networking on [Cisco Communities](https://community.cisco.com/t5/sd-wan-and-cloud-networking/bd-p/discussions-sd-wan)
* Cisco [Digital Learning](https://digital-learning.cisco.com)
* SD-WAN Sandboxes and Learning tracks on [DevNet](https://developer.cisco.com/sdwan/)
* SD-WAN Design [Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/SDWAN/cisco-sdwan-design-guide.html)
* SD-WAN Deployment [Guide](https://www.cisco.com/c/dam/en/us/td/docs/solutions/CVD/SDWAN/SD-WAN-End-to-End-Deployment-Guide.pdf)
* SDWAN Troubleshooting [Guide](https://www.cisco.com/c/en/us/support/docs/routers/sd-wan/214509-troubleshoot-control-connections.html#anc13)


## 1.0 Overview

Below summarises the devices found in each functional zone and also the key components of the virtual lab:
* __Control-plane__: vSmart - establishes OMP sessions between itself and edge devices, facilitates fabric discovery
* __Management-plane__: vManage - provisioning, configuration templating, visibility for day-1 and day-2 ops, REST API interface for programming and automation. vBond orchestrates the discovery of vSmart and vManage devices for WAN edge devices
* __Data-plane__: 6 x vEdge and cEdge devices in Auckland, Hamilton, Wellington, Christchurch
* __Transport__: Direct Internet and MPLS WAN


##### Physical Topology
![Physical Topology](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Physical_Design_2.png?raw=true)

Let's step through creating the topology in a virtual environment.

## 1.2 Lab components

Appliance | Image name | Version | Memory | CPU | Disk | Qty
------------ | ------------- | ------------- | ------------- | ------------- | ------------- | -------------
vManage | viptela-vmanage-20.3.3.1-genericx86-64.qcow2 | 20.3.3.1 | 32768MB | 1vCPUs | 30G | 1
vSmart | viptela-smart-20.3.3-genericx86-64.qcow2 | 20.3.3 | 4096MB | 1vCPUs | - | 1
vBond | viptela-edge-20.3.3-genericx86-64.qcow2 | 20.3.3 | 2048MB | 2vCPUs | - | 1
vEdge | viptela-edge-20.3.3-genericx86-64.qcow2 | 20.3.3 | 2048MB | 1vCPUs | - | 6
csr1000v | csr1000v-universalk9.17.03.03-serial.qcow2 | 17.03.03 | 3072MB | 1vCPUs | - | 1
Ubuntu Desktop | Ubuntu 20.10 (64bit).vmdk | 20.10 | 1024MB | 1vCPUs | - | 3


## 1.3 Addressing

*  __VPN 0 Transport VPN__: WAN transport and control plane traffic.  Contains all interfaces except Management.
*  __VPN 512 Management VPN__: Carries out of band management traffic and interfaces

Subnet | Mask | Purpose
------------ | ------------- | -------------
[192.168.200.0] | 255.255.255.0 | VPN 512 Mgmt OOB
10.254.254.0 | 255.255.255.0 | VPN 0
10.254.253.0 | 255.255.255.0 | VPN 512 Mgmt In-Band
10.100.0.0 | 255.255.0.0 | Direct Internet
10.200.0.0 | 255.255.0.0 | WAN



## 1.4 Interfaces

### 1.4.1 Control and Management Networks

Hostname | Interface | Address | Network
------------ | ------------- | ------------- | -------------
vmanage | eth0 | 192.168.200.42 | mgmt-oob
vmanage | eth1 | 10.254.253.1| vpn 512
vmanage | eth2 | 10.254.254.1 | vpn 0
vbond | eth0 | 10.254.253.3 | vpn 512
vbond | eth1 | 10.254.254.3 | vpn 0
vsmart | eth2 | 10.254.254.2 | vpn 0


### 1.4.2 Tranport Networks

Hostname | Interface | Address | Network
------------ | ------------- | ------------- | -------------
border-router | eth1 | 10.254.254.254 | vpn 512
border-router | eth2 | 10.100.0.1 | DIA
border-router | eth2 | 10.200.0.1 | WAN


### 1.4.3 Edge and Branch Networks

Hostname | Interface | Address | Network
------------ | ------------- | ------------- | -------------
akl-vedge-1 | g0/0 | 10.100.0.10 | DIA
akl-vedge-1| g0/1 | 10.200.0.10 | WAN
akl-vedge-2 | g0/0 | 10.100.0.11 | DIA
akl-vedge-2 | g0/1 | 10.200.0.11 | WAN
wgn-vedge-1 | g0/0 | 10.200.0.20 | DIA
wgn-vedge-1| g0/1 | 10.100.0.20 | WAN
wgn-vedge-2| g0/1 | 10.100.0.21 | DIA
wgn-vedge-2 | g0/0 | 10.200.0.21 | WAN
hml-vedge-1 | g0/0 | 10.100.0.30 | DIA
hml-vedge-1 | g0/1 | 10.200.0.30 | WAN
chc-vedge-1 | g0/0 | 10.100.0.40 | DIA
chc-vedge-1| g0/1 | 10.200.0.40 | WAN


## 1.5 Device Configurations

### 1.5.1 Provisioning and Controller Profile
In the Plug and Play Connect portal on software.cisco.com:
* Configure the controller profile as shown:
* Add software devices
* Download the vedge provisioning file that we'll upload to vManage and use to authenticate vedge devices

##### Controller Profile
![Controller Profile](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Plug_Play_Controller_profile.png?raw=true)

##### Download the Provisioning File
![Provisioning File](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Provisioning_FIle.png?raw=true)


### 1.6.1 vManage

As per 1.4.1, eth2 maps to VPN 0 which is used for control traffic, eth0 is used for "out of band" lab management and eth1 for management inband, both map to VPN 512 for management

#### System
```
vmanage# show running-config system
system
 host-name             vmanage
 system-ip             1.1.1.1
 site-id               1000
 admin-tech-on-failure
 sp-organization-name  nz-se-lab
 organization-name     nz-se-lab
 clock timezone Pacific/Auckland
 vbond 10.254.254.3
 !
 ntp
  server nz.pool.ntp.org
   version 4
  exit
 !
```

#### Control - vpn 0
```
vmanage# sh run vpn 0
vpn 0
 interface eth2
  ip address 10.254.254.1/24
  tunnel-interface
   allow-service dhcp
   allow-service dns
   allow-service icmp
   no allow-service sshd
   no allow-service netconf
   no allow-service ntp
   no allow-service stun
   allow-service https
  !
  no shutdown
 !
 ip route 0.0.0.0/0 10.254.254.254
!
```

#### Management - vpn 512

```
vmanage# show running-config vpn 512
vpn 512
 interface eth0
  ip dhcp-client
  no shutdown
 !
 interface eth1
  ip address 10.254.253.1/24
  no shutdown
 !
 ip route 192.168.0.0/16 192.168.200.1 <- Route all home network subnets to firewall
!
```

Verify that an address has been assigned to eth0 and that it's reachable through a browser.  In my case https://192.168.200.42.

#### Initial Configuration Settings

From __Administration > Settings__, enter the organization name (must much plug and play controller profile) and vbond VPN0 IP address
![Configuration Settings](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Administration.Settings.png?raw=true)

The vbond address needs to be configued under **both** the cli and gui as above for successful vedge registration.

##### Generate the keys for certificates

Enter __vshell__ mode on vmanage and run the following:
```
openssl genrsa -out SDWAN.key 2048
openssl req -x509 -new -nodes -key SDWAN.key -sha256 -days 2000 \
        -subj "/C=NZ/ST=NZ/L=NZ/O=nz-se-lab/CN=SD-WAN" \
        -out SDWAN.pem
```
Use __cat SDWAN.pem__, to copy the certificate to Administration > Settings > Controll > Enterprise Root Certificate
![Enterprise Root Cert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Enterprise_Root_certificate.png?raw=true)

Browse to __https://vmanage-ip-address/dataservice/system/device/sync/rootcertchain__, this will resync the vmanage database via an API call.

```
openssl x509 -req -in vManage.csr -CA SDWAN.pem -CAkey SDWAN.key -CAcreateserial -out vManage.crt -days 2000 -sha256
cat vManage.crt
```

Under Configuration > Certificates > Controllers, select vManage and "Generate CSR", copy the CSR content.
![Generate CSR](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Generate_CSR.png?raw=true)

In the vshell on vmanage use VIM to paste the CSR content into vManage.csr and save.

Sign the CSR as below:
```
openssl x509 -req -in vManage.csr -CA SDWAN.pem -CAkey SDWAN.key -CAcreateserial -out vManage.crt -days 2000 -sha256
```
Use **cat** to copy vManage.crt content, and install through Configuration > Certificates > Controllers and select vManage.

Paste vManage.crt content and "install" as below.

![Install Cert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Install_vManage_cert.png?raw=true)



### 1.6.2 vBond

#### system
```
vbond# show running-config system
system
 host-name               vbond
 system-ip               1.1.1.3
 site-id                 1000
 admin-tech-on-failure
 no route-consistency-check
 organization-name       nz-se-lab
 clock timezone Pacific/Auckland
 vbond 10.254.254.3 local vbond-only
 !
  ntp
   server nz.pool.ntp.org
    version 4
   exit
  !
```
The vbond [10.254.254.3] __local__ command designates that the device functions as a vbond vs vedge.

#### Control - VPN 0
**Note** - you will probably need to disable the "tunnel-interface" to add the vbond controller to vManage
```
vbond# show running-config vpn 0
vpn 0
 interface ge0/1
  ip address 10.254.254.3/24
  tunnel-interface
   encapsulation ipsec
   no allow-service bgp
   allow-service dhcp
   allow-service dns
   allow-service icmp
   no allow-service sshd
   no allow-service netconf
   no allow-service ntp
   no allow-service ospf
   no allow-service stun
   allow-service https
  !
  no shutdown
 !
 ip route 0.0.0.0/0 10.254.254.254
```

#### Management - VPN 512
```
vbond# show running-config vpn 512
vpn 512
 interface eth0
  ip address 172.16.1.3/24
  ipv6 dhcp-client
  no shutdown
 !
!
```

#### vBond Certificate Install
Either use **cat** and to copy the contents of **SDWAN.pem** and **SDWAN.key** from vmanage in vshell mode to vbond using **vim**, or use SCP to copy the .pem and .key files from vmanage to vbond.

In vManage add vBond under Configuration > Controllers > Add Controller.  Use the VPN 0 (Control interface) IP address 10.254.254.3.
![Add vBond](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Add_vBond.png?raw=true)

Under Configuration > Certificates > Controllers > vBond, click the three dots ... and "View CSR"

In vshell mode, use VIM to paste the contents into vBond.csr

```
openssl x509 -req -in vBond.csr -CA SDWAN.pem -CAkey SDWAN.key -CAcreateserial -out vBond.crt -days 2000 -sha256
```
Copy the content of vBond.crt (__cat vBond.crt__) and paste to **Configuration > Certificates > Controllers > Select vBond** and "Install Certificate":
![vBond Cert](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/vbond_cert.png?raw=true)

### 1.6.3 vSmart

#### system
```
vsmart# show running-config system
system
 host-name             vsmart
 system-ip             1.1.1.2
 site-id               1000
 admin-tech-on-failure
 organization-name     nz-se-lab
 clock timezone Pacific/Auckland
 vbond 10.254.254.3
 !
 ntp
  server nz.pool.ntp.org
   version 4
  exit
 !
```

#### VPN 0
**Note** - you will probably need to disable the "tunnel-interface" to add the vsmart controller to vManage
```
vsmart# show running-config vpn 0
vpn 0
 interface eth1
  ip address 10.254.254.2/24
  tunnel-interface
   allow-service dhcp
   allow-service dns
   allow-service icmp
   no allow-service sshd
   no allow-service netconf
   no allow-service ntp
   no allow-service stun
  !
  no shutdown
 !
 ip route 0.0.0.0/0 10.254.254.254
```

#### VPN 512
```
vsmart# show running-config vpn 512
vpn 512
 interface eth0
  ip address 172.16.1.2/24
  no shutdown
 !
!
```

#### vSmart Certificate Install

* Use **cat** and to copy the contents of **SDWAN.pem** and **SDWAN.key** from vmanage in vshell mode to vSmart using **vim**, or use SCP to copy the .pem and .key files from vmanage to vsmart.
* In vManage add vSmart under Configuration > Controllers > Add Controller.  Use the VPN 0 (Control interface) IP address 10.254.254.2.
* Under Configuration > Certificates > Controllers > vSmart, click the three dots ... and "View CSR"
* In vshell mode, use VIM to paste the contents into vSmart.csr
* ```
openssl x509 -req -in vSmart.csr -CA SDWAN.pem -CAkey SDWAN.key -CAcreateserial -out vSmart.crt -days 2000 -sha256
```
* Copy the contents of vSmart.crt (__cat vSmart.crt__) and paste to **Configuration > Certificates > Controllers > Select vSmart** and "Install Certificate":

### 1.6.4 vEdge

#### Certificates
* In vshell mode on vManage, copy SDWAN.pem to vEdge
  ```
  vmanage:~$ cat SDWAN.pem
  -----BEGIN CERTIFICATE-----
     *********************
  -----END CERTIFICATE-----
  ```
* Paste the cert contents into the vedge in vshell mode **vim SDWAN.pem**
* Install the root certificate
  ```
  request root-cert-chain install /home/admin/SDWAN.pem
  ```
* Verify certificate has been Installed
  ```
  akl-vedge-1# show certificate root-ca-cert | more
  Certificate:
      Data:
          Version: 3 (0x2)
          Serial Number:
              18:da:d1:9e:26:7d:e8:bb:4a:21:58:cd:cc:6b:3b:4a
  ```
* Upload the WAN Edge List generated in section 1.5.1 to vManage - Configuration > Devices
  ![WAN Edge List](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/WAN%20Edge%20List.png?raw=true)
* Select a vedge cloud device and **Generate Bootstrap**, note the UUID and OTP for the next step
  ![Generate Bootstrap](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/Generate_Bootstrap.png?raw=true)
* Return to the vedge device, exit from vshell mode and register with the UUID and OTP shown above
  ```
  akl-vedge-1# request vedge-cloud activate chassis-number **************** token ***************
  ```

#### System
```
akl-vedge-1# show running-config system
system
 host-name               akl-vedge-1
 system-ip               2.2.2.1
 site-id                 1
 admin-tech-on-failure
 no route-consistency-check
 organization-name       nz-se-lab
 clock timezone Pacific/Auckland
 vbond 10.254.254.3
 !
 ntp
  server nz.pool.ntp.org
   version 4
  exit
 !
 ```

#### Control VPN 0

 ```
 vpn 0
 interface ge0/0
  ip address 10.100.0.10/16
  tunnel-interface
   encapsulation ipsec
   no allow-service bgp
   allow-service dhcp
   allow-service dns
   allow-service icmp
   no allow-service sshd
   no allow-service netconf
   no allow-service ntp
   no allow-service ospf
   no allow-service stun
   allow-service https
  !
  no shutdown
 !
 interface ge0/1
  ip address 10.200.0.10/24
  no shutdown
 !
 ip route 0.0.0.0/0 10.100.0.1
 ```

## 1.7 Verification

From the vManage GUI, all devices are green and "up":
![vManage](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/vManage.png?raw=true)

WAN edge status is "reachable":
![WAN Edge Status](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/WAN_Edge_status.png?raw=true)

Clock and timezone is synchronised:
```
vmanage# show ntp associations

                                                           LAST             
IDX  ASSOCID  STATUS  CONF  REACHABILITY  AUTH  CONDITION  EVENT     COUNT  
----------------------------------------------------------------------------
1    40044    961a    yes   yes           none  sys.peer   sys_peer  1      

vmanage# show clock
Mon May 10 17:09:49 NZST 2021
```

Certificate is installed, organisation name is correct, vbond connection up, WAN interfaces up:
```
vmanage# show control local-properties
personality                       vmanage
sp-organization-name              nz-se-lab
organization-name                 nz-se-lab
root-ca-chain-status              Installed

certificate-status                Installed
certificate-validity              Valid
certificate-not-valid-before      May 09 21:46:52 2021 GMT
certificate-not-valid-after       Oct 30 21:46:52 2026 GMT

dns-name                          10.254.254.3
site-id                           1000
domain-id                         0
protocol                          dtls
tls-port                          23456
system-ip                         1.1.1.1
number-vbond-peers                1

INDEX   IP                                      PORT
-----------------------------------------------------
0       10.254.254.3                            12346  

number-active-wan-interfaces      2

                                PUBLIC          PUBLIC PRIVATE         PRIVATE                                 PRIVATE                               LAST
INSTANCE             INTERFACE  IPv4            PORT   IPv4            IPv6                                    PORT    VS/VM  COLOR            STATE CONNECTION
---------------------------------------------------------------------------------------------------------------------------------------------------------------
0        eth2       10.254.254.1    12346  10.254.254.1    ::                                      12346     1/0   default           up     0:00:00:04
1        eth2       10.254.254.1    12446  10.254.254.1    ::                                      12446     0/0   default           up     0:00:00:04
```
From vBond - validate control connectivity to vmanage, vsmart and vedges:
```
vbond# show orchestrator connections
                                                                                     PEER                      PEER                                                                            
         PEER     PEER     PEER             SITE        DOMAIN      PEER             PRIVATE  PEER             PUBLIC                                   ORGANIZATION                           
INSTANCE TYPE     PROTOCOL SYSTEM IP        ID          ID          PRIVATE IP       PORT     PUBLIC IP        PORT    REMOTE COLOR     STATE           NAME                    UPTIME         
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
0        vedge    dtls     2.2.2.1          1           1           10.100.0.10      12366    10.100.0.10      12366   default          up              nz-se-lab               0:04:18:39     
0        vedge    dtls     2.2.2.2          1           1           10.100.0.11      12346    10.100.0.11      12346   default          up              nz-se-lab               0:04:18:37     
0        vedge    dtls     2.2.2.3          2           1           10.100.0.20      12366    10.100.0.20      12366   default          up              nz-se-lab               0:04:18:36     
0        vedge    dtls     2.2.2.4          2           1           10.100.0.21      12366    10.100.0.21      12366   default          up              nz-se-lab               0:04:18:35     
0        vedge    dtls     2.2.2.5          3           1           10.100.0.30      12346    10.100.0.30      12346   default          up              nz-se-lab               0:00:01:28     
0        vedge    dtls     2.2.2.6          4           1           10.100.0.40      12346    10.100.0.40      12346   default          up              nz-se-lab               0:00:00:06     
0        vsmart   dtls     1.1.1.2          1000        1           10.254.254.2     12346    10.254.254.2     12346   default          up              nz-se-lab               0:06:40:17     
0        vsmart   dtls     1.1.1.2          1000        1           10.254.254.2     12446    10.254.254.2     12446   default          up              nz-se-lab               0:06:40:16     
0        vmanage  dtls     1.1.1.1          1000        0           10.254.254.1     12346    10.254.254.1     12346   default          up              nz-se-lab               0:06:49:14     
0        vmanage  dtls     1.1.1.1          1000        0           10.254.254.1     12446    10.254.254.1     12446   default          up              nz-se-lab               0:06:49:31
```
##### Verify Connectivity
vbond to Border Gateway:
```
vbond# ping 10.254.254.254 vpn 0 count 3
Ping in VPN 0
PING 10.254.254.254 (10.254.254.254) 56(84) bytes of data.
64 bytes from 10.254.254.254: icmp_seq=1 ttl=255 time=107 ms
64 bytes from 10.254.254.254: icmp_seq=2 ttl=255 time=25.6 ms
64 bytes from 10.254.254.254: icmp_seq=3 ttl=255 time=64.8 ms

--- 10.254.254.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 25.620/65.906/107.215/33.319 ms
```
vbond to vManage:
```
vbond# ping 10.254.254.1 vpn 0 count 3  
Ping in VPN 0
PING 10.254.254.1 (10.254.254.1) 56(84) bytes of data.
64 bytes from 10.254.254.1: icmp_seq=1 ttl=64 time=12.3 ms
64 bytes from 10.254.254.1: icmp_seq=2 ttl=64 time=14.3 ms
64 bytes from 10.254.254.1: icmp_seq=3 ttl=64 time=15.2 ms

--- 10.254.254.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 12.355/14.005/15.291/1.229 ms
```
vbond to vsmart:
```
vbond# ping 10.254.254.2 vpn 0 count 3
Ping in VPN 0
PING 10.254.254.2 (10.254.254.2) 56(84) bytes of data.
64 bytes from 10.254.254.2: icmp_seq=1 ttl=64 time=20.1 ms
64 bytes from 10.254.254.2: icmp_seq=2 ttl=64 time=12.2 ms
64 bytes from 10.254.254.2: icmp_seq=3 ttl=64 time=19.4 ms

--- 10.254.254.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 12.206/17.282/20.199/3.602 ms
```
vbond to akl-vedge-1:
```
vbond# ping 10.100.0.10 vpn 0 count 3
Ping in VPN 0
PING 10.100.0.10 (10.100.0.10) 56(84) bytes of data.
64 bytes from 10.100.0.10: icmp_seq=1 ttl=63 time=16.2 ms
64 bytes from 10.100.0.10: icmp_seq=2 ttl=63 time=16.2 ms
64 bytes from 10.100.0.10: icmp_seq=3 ttl=63 time=14.2 ms

--- 10.100.0.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 14.211/15.556/16.240/0.962 ms
```
vbond to wgn-vedge-1:
```
vbond# ping 10.100.0.20 vpn 0 count 3
Ping in VPN 0
PING 10.100.0.20 (10.100.0.20) 56(84) bytes of data.
64 bytes from 10.100.0.20: icmp_seq=1 ttl=63 time=2.27 ms
64 bytes from 10.100.0.20: icmp_seq=2 ttl=63 time=13.4 ms
64 bytes from 10.100.0.20: icmp_seq=3 ttl=63 time=12.2 ms

--- 10.100.0.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 2.275/9.319/13.435/5.006 ms
```
vbond to vbond to hml-vedge-1:
```
vbond# ping 10.100.0.30 vpn 0 count 3
Ping in VPN 0
PING 10.100.0.30 (10.100.0.30) 56(84) bytes of data.
64 bytes from 10.100.0.30: icmp_seq=1 ttl=63 time=19.2 ms
64 bytes from 10.100.0.30: icmp_seq=2 ttl=63 time=18.2 ms
64 bytes from 10.100.0.30: icmp_seq=3 ttl=63 time=18.3 ms

--- 10.100.0.30 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 18.238/18.606/19.234/0.472 ms
```
vbond to chc-vedge-1:
```
vbond# ping 10.100.0.40 vpn 0 count 3
Ping in VPN 0
PING 10.100.0.40 (10.100.0.40) 56(84) bytes of data.
64 bytes from 10.100.0.40: icmp_seq=1 ttl=63 time=20.2 ms
64 bytes from 10.100.0.40: icmp_seq=2 ttl=63 time=19.3 ms
64 bytes from 10.100.0.40: icmp_seq=3 ttl=63 time=19.3 ms

--- 10.100.0.40 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 19.318/19.631/20.219/0.416 ms
```

Connectivity successful, validate control connections between vedge and vbond
```
vedge-6# show control connections
                                                                                       PEER                                          PEER                                          CONTROLLER
PEER    PEER PEER            SITE       DOMAIN PEER                                    PRIV  PEER                                    PUB                                           GROUP      
TYPE    PROT SYSTEM IP       ID         ID     PRIVATE IP                              PORT  PUBLIC IP                               PORT  LOCAL COLOR     PROXY STATE UPTIME      ID         
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
vsmart  dtls 1.1.1.2         1000       1      10.254.254.2                            12346 10.254.254.2                            12346 default         No    up     0:00:21:04  0           
vbond   dtls 0.0.0.0         0          0      10.254.254.3                            12346 10.254.254.3                            12346 default         -     up     0:00:21:04  0           
vmanage dtls 1.1.1.1         1000       0      10.254.254.1                            12346 10.254.254.1                            12346 default         No    up     0:00:21:04  0
```
## Lessons Learned
* Watch for ommitted or extra characters when copying certificates between devices - this caught me out a few times
* You may need to disable encapsulation on the interface for devices to register with vmanage and re-enable once registered. You may need to toggle on/off depending on the order of operations
* Set the timezone / clock on all devices and ensure they're in sync
* Organization name is important and must match between Plug and Play / Smart License portal and vmanage
* Ensure the vBond IP address matches the controller address defined in the plug and play portal, otherwise vEdges won't register
* Use "show control conections-history detail" if and when troubleshooting bring up of devices and pay close attention to any error codes, cross reference any error codes with the troubleshooting guide linked above.  
* Reminder - management interfaces in VPN 512, control/transport in VPN 0.  This caught me out as below.


##### An Example

An error I encountered was "VM_TMO" and from "show control connections-history detail":
  ```
  "state               connect [Local Err: ERR_(D)TLS_CONN_FAIL] [Remote Err: NO_ERROR]"
  ```

  Regular pings were completing but rapid pings weren't:

  ```
  vedge-1# ping 10.254.254.1 vpn 0
  Ping in VPN 0
  PING 10.254.254.1 (10.254.254.1) 56(84) bytes of data.
  64 bytes from 10.254.254.1: icmp_seq=2 ttl=63 time=1.08 ms
  64 bytes from 10.254.254.1: icmp_seq=4 ttl=63 time=0.885 ms
  64 bytes from 10.254.254.1: icmp_seq=5 ttl=63 time=1.30 ms

  vedge-1# ping 10.254.254.1 vpn 0 count 1000 rapid
  Ping in VPN 0
  !.!....!.!..!.!.!.!..!.!^C
  ```
  I'd put my management interface and route in the wrong vpn (0)  - Moving the management interface and static route from vpn 0 to vpn 512 resolved.

  ```
  --- 10.254.254.1 statistics ---
  24 packets transmitted, 10 received, 59% packet loss
  vedge-1# ping 10.254.254.3 vpn 0 count 1000 rapid
  Ping in VPN 0
  !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!^C
  ```
The [Troubleshooting Guide](https://www.cisco.com/c/en/us/support/docs/routers/sd-wan/214509-troubleshoot-control-connections.html#anc11) describes each of these error codes and steps to resolve.

## Wrap Up
Like anything, practice and repetition and familiarity with order of operations will help a lot.  It took me a couple of hours to get my first vEdges registered with vManage and about 30 minutes to register the remaining.

This wraps up the SDWAN lab deployment, we'll cover vManage APIs and Visualisations with Grafana in the next update.
