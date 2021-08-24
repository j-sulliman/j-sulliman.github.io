---
layout: default
title: Intersight - Automation of Delivery Documentation
parent: NetDevOps -  all other articles
nav_order: last
last_modified_date: 2021-07-19 08:00:00 +1200
---

[![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/j-sulliman/iabg)

# iABG - Intersight AsBuilt Generator

Autmatic generation of base AsBuilt documents for intersight.

Reports on:

* Firmware
* Blade and rackmount servers
* Hyperflex
* FC and Ethernet Interfaces
* Service Profiles
* Management Interfaces

Overview:

* Uses the intersight_rest API to pull data from intersight
* Django and Bootrap4 provide the web front end to receive user input (intersight API keys) by way of a form
* Data retrieved from API call is stored in a dataframe using the pandas module
* Dataframes then written to an excel workbook
* Word report generated using the python-docx package
* drawio topology template included for reference also

## Demo (click to view)
[![IMAGE ALT TEXT](http://img.youtube.com/vi/EjcmM4tDglg/0.jpg)](http://www.youtube.com/watch?v=EjcmM4tDglg "iABG")

Enter your intersight API keys using the form on the left, documents will be output for download on the right:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/iABG_Example.PNG?raw=true)

Word Template:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/word_example.PNG?raw=true)

Excel Workbook:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/excel.PNG?raw=true)

Drawio Diagram:
![alt text](https://github.com/j-sulliman/j-sulliman.github.io/blob/master/images/is_diagram.PNG?raw=true)

## Setup

```powershell
python -m venv venv

.\venv\Scripts\activate
git init
git pull https://github.com/j-sulliman/iabg.git

Install the required dependencies:
cd is
pip install -r .\requirements.txt

Installing collected packages: urllib3, six, idna, charset-normalizer, certifi, sqlparse, requests, pytz, python-dateutil, pycryptodomex, pycryptodome, numpy, lxml, et-xmlfile, asgiref, python-docx, pandas, openpyxl, intersight-rest, Django
Successfully installed Django-3.2.6 asgiref-3.4.1 certifi-2021.5.30 charset-normalizer-2.0.4 et-xmlfile-1.1.0 idna-3.2 intersight-rest-1.1.7 lxml-4.6.3 numpy-1.21.2 openpyxl-3.0.7 pandas-1.3.2 pycryptodome-3.10.1 pycryptodomex-3.10.1 python-dateutil-2.8.2 python-docx-0.8.11 pytz-2021.1 requests-2.26.0 six-1.16.0 sqlparse

```

### Start Django Server

```python
python .\manage.py runserver 0.0.0.0:8080

Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
August 24, 2021 - 10:29:29
Django version 3.2.6, using settings 'is.settings'
Starting development server at http://0.0.0.0:8080/
Quit the server with CTRL-BREAK.
```
