---
layout: default
title: Part 1 - Instrumenting a dotnet6 container with appdynamics
nav_order: last
last_modified_date: 2021-10-12 08:00:00 +1200
published: true
perent: Observability
permalink: docs/observability/dotnet-containers
---


# 1.1 Introduction

For this article I thought I'd document instrumenting a basic dotnet6 application with a sql backend, hosted as a docker container using the sample eShopOnWeb application, to hopefully help others who like me may be interested in learning the basics of dotnet6, docker and instrumenting with a AppDynamics APM agent.


# 1.2 References
*  [Install the .NET Agent for Linux in Containers](https://docs.appdynamics.com/appd/22.x/latest/en/application-monitoring/install-app-server-agents/net-agent/net-agent-for-linux/net-agent-for-linux-container-installation/install-the-net-agent-for-linux-in-containers#id-.Installthe.NETAgentforLinuxinContainersv22.2-dockerfile)
*  [eShopOnWeb GitHub Repo](https://github.com/dotnet-architecture/eShopOnWeb)
*  [Apache JMeter](https://jmeter.apache.org/download_jmeter.cgi)

---

# 1.3 Download the eShop Web github repo


Create the directory for our sandbox:
```shell
$ mkdir dotnet6-sandbox
```

Pull the source code for the eShopOnWeb application:
```shell
$ cd dotnet6-sandbox/
$ git init
$ git pull https://github.com/dotnet-architecture/eShopOnWeb.git
```

# 1.4 Edit the docker files to install the AppDynamics Agent
As noted in the installation documents - download and unzip the .NET Agent for linux:

```shell
$ curl -L -O -H "Authorization: Bearer < blah >" "https://download.appdynamics.com/download/prox/download-file/dotnet-core/22.10.0/AppDynamics-DotNetCore-linux-x64-22.10.0.zip"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0
100 6845k  100 6845k    0     0  1846k      0  0:00:03  0:00:03 --:--:-- 8504k

$ unzip AppDynamics-DotNetCore-linux-x64-22.10.0.zip -d AppDynamics-DotNetCore-linux-x64
```

Edit the dockerfile with the information needed to connect to the AppDynamics controller:
```shell
$ cat src/Web/Dockerfile
# RUN ALL CONTAINERS FROM ROOT (folder with .sln file):
# docker-compose build
# docker-compose up
#
# RUN JUST THIS CONTAINER FROM ROOT (folder with .sln file):
# docker build --pull -t web -f src/Web/Dockerfile .
#
# RUN COMMAND
#  docker run --name eshopweb --rm -it -p 5106:5106 web
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

COPY *.sln .
COPY . .
WORKDIR /app/src/Web
RUN dotnet restore

RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
WORKDIR /app
COPY --from=build /app/src/Web/out ./

# Optional: Set this here if not setting it from docker-compose.yml
# ENV ASPNETCORE_ENVIRONMENT Development

ENTRYPOINT ["dotnet", "Web.dll"]
RUN mkdir /opt/appdynamics/
COPY AppDynamics-DotNetCore-linux-x64/ /opt/appdynamics/
ENV CORECLR_PROFILER="{57e1aa68-2229-41aa-9931-a6e93bbc64d8}"
ENV CORECLR_ENABLE_PROFILING=1
ENV CORECLR_PROFILER_PATH="/opt/appdynamics/libappdprofiler.so"
ENV APPDYNAMICS_AGENT_APPLICATION_NAME="z_jamie_dotnet_eShopOnWeb"
ENV APPDYNAMICS_AGENT_TIER_NAME="Web_Front_End"
ENV APPDYNAMICS_AGENT_ACCOUNT_NAME="apjsales2"
ENV APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY="your-access-key"
ENV APPDYNAMICS_CONTROLLER_HOST_NAME="example.saas.appdynamics.com"
ENV APPDYNAMICS_CONTROLLER_PORT=443    
ENV APPDYNAMICS_CONTROLLER_SSL_ENABLED=true   
ENV APPDYNAMICS_AGENT_REUSE_NODE_NAME=true
ENV APPDYNAMICS_AGENT_REUSE_NODE_NAME_PREFIX="jamies-instance" 
ENV LD_LIBRARY_PATH=/opt/appdynamics/dotnet
# variables required to send transaction analytics data
ENV APPDYNAMICS_ANALYTICS_HOST_NAME="example.saas.appdynamics.com"
ENV APPDYNAMICS_ANALYTICS_PORT=443
ENV APPDYNAMICS_ANALYTICS_SSL_ENABLED=true
```

Edit docker-compuse file, in my case I needed to:
* Hard code DNS server in the hosts /etc/resolve.conf file (I only allow DNS to Cisco umbrella DNS servers on my home network)
* Set the network mode to bridge to allow outside (i.e 443 to the AppD controller)
* Setting the network mode to bridge allows access to the outside network, but breaks connections between containers - set the "links" parameter to allow the front end container to talk to the sql-server container

```shell
$ cat docker-compose.yml 
version: '3.4'

services:
  eshopwebmvc:
    image: ${DOCKER_REGISTRY-}eshopwebmvc
    build:
      context: .
      dockerfile: src/Web/Dockerfile
    links:
      - "sqlserver"
    depends_on:
      - "sqlserver"
    dns:
      - 208.67.220.220
      - 208.67.222.222
      - 8.8.8.8
    network_mode: "bridge"
  eshoppublicapi:
    image: ${DOCKER_REGISTRY-}eshoppublicapi
    build:
      context: .
      dockerfile: src/PublicApi/Dockerfile
    depends_on:
      - "sqlserver"
    dns:
      - 208.67.220.220
      - 208.67.222.222
      - 8.8.8.8
    network_mode: "bridge"
  sqlserver:
    image: mcr.microsoft.com/azure-sql-edge
    ports:
      - "1433:1433"
    environment:
      - SA_PASSWORD=@someThingComplicated1234
      - ACCEPT_EULA=Y
    dns:
      - 208.67.220.220
      - 208.67.222.222
      - 8.8.8.8
    network_mode: "bridge"
```


The directory structure should resemble below - note the locations of the AppDynamics agent directory, docker-compose.yml and Dockerfile:
```shell
.
├── AppDynamics-DotNetCore-linux-x64
│   ├── AppDynamics.Agent.netstandard.dll
│   ├── AppDynamics.Agent.OtelSDK.dll
│   ├── AppDynamicsConfig.json.template
│   ├── libappdprofiler_glibc.so
│   ├── libappdprofiler_musl.so
│   ├── libappdprofiler.so
│   └── README.md
├── AppDynamics-DotNetCore-linux-x64-22.10.0.zip
├── CodeCoverage.runsettings
├── docker-compose.bak
├── docker-compose.dcproj
├── docker-compose.override.yml
├── docker-compose.yml <-- Docker-compose referenced above
├── eShopOnWeb.sln
├── global.json
├── LICENSE
├── README.md
├── src
│   ├── ApplicationCore
│   ├── BlazorAdmin
│   ├── BlazorShared
│   ├── Infrastructure
│   ├── PublicApi
│   └── Web
│       ├── Dockerfile <-- DockerFile referenced above
└── tests
```

# 1.4 Build and Start the Docker Containers
Let's try building the docker images:

```shell
$ sudo docker-compose build
$ sudo docker-compose up
```
Check the status of our containers:
```shell
$ sudo docker ps
[sudo] password for jamie: 
CONTAINER ID   IMAGE                              COMMAND                  CREATED         STATUS         PORTS                                                 NAMES
5626af813ddc   eshopwebmvc                        "dotnet Web.dll"         2 minutes ago   Up 2 minutes   0.0.0.0:5106->80/tcp, :::5106->80/tcp                 dotnet6-sandbox_eshopwebmvc_1
5243ba9f4447   eshoppublicapi                     "dotnet PublicApi.dll"   2 minutes ago   Up 2 minutes   443/tcp, 0.0.0.0:5200->80/tcp, :::5200->80/tcp        dotnet6-sandbox_eshoppublicapi_1
de9f3931bfb8   mcr.microsoft.com/azure-sql-edge   "/opt/mssql/bin/perm…"   2 minutes ago   Up 2 minutes   1401/tcp, 0.0.0.0:1433->1433/tcp, :::1433->1433/tcp   dotnet6-sandbox_sqlserver_1
```
Check the network details of our front end web-site container:
```shell
$ sudo docker inspect 5626af813ddc | grep IPAdd
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.4",
                    "IPAddress": "172.17.0.4",
```
You should be able to browse to the site on the above IP address:

![Front-End](https://user-images.githubusercontent.com/782127/88414268-92d83a00-cdaa-11ea-9b4c-db67d95be039.png)

# 1.5 Using Apache JMeter to Generate Traffic

Here we use the free Apache JMeter utility to generate traffic to the eshop website.

Download using the link above (note the Java 1.8+ requirement) and run:

```shell
$ cd ~/Downloads/apache-jmeter-5.5/
$ cd bin/
$ ./jmeter.sh
```

I've setup some basic HTTP Requests as below - apache-jmeter.  (Note the server name or IP and Paths fields):

![Front-End](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/assets/images/apache-jmeter.png?raw=true)


All going well, we'll now see the application flow map and business transactions populate in the controller

Flow Map:
![Flow-Map](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/assets/images/dotnet-flowmap.png?raw=true)

Business Transactions:
![Business-Transactions](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/assets/images/dotnet-bts.png?raw=true)
