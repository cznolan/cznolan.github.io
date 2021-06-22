---
layout: post
title: "Cisco Device Backup With Nornir + NAPALM"
date: 2021-06-13 20:30:00 +0800
tags: cisco backup nornir napalm netmiko xe xr nx-os ios github
author: cznolan
---
As I regularly use Python when working with files, and Netmiko when programmatically working with Cisco devices, I wanted to try a Python-based device configuration backup solution. I have not yet had the opportunity to test Nornir and NAPALM, so I decided to give them a shot rather than sticking with purely Netmiko.

This guide will run through the basic setup of Nornir and writing backup files to both a local disk and a GitHub repository. I am going to gloss over the specifics of how Nornir and NAPALM work, as there is quite a lot of documentation available.

My setup is as follows:
* CentOS 8.4.2105 & Windows 10 21H1
* Python 3.8.6
* NAPALM 3.3.0
* Netmiko 3.4.0
* Nornir 3.1.0
  * NAPALM Plugin 0.1.2
  * Utils Plugin 0.1.2
* IOS XE device running 16.12.4 (connect via SSH)
* IOS XR device running 7.0.1 (connect via NETCONF)
* NX-OS device running 9.3.3 (connect via NX-API)

## What Are Nornir and Napalm?

Nornir is an Python-based automation framework that handles the device inventory and keeps track of the collected data. It provides functionality similar to that of Ansible.

NAPALM is a Python-based abstraction layer, that allows actions to be applied against devices running various network operating systems based on intent rather than on vendor-specific commands. It provides abstraction functionality similar to that of Ansible modules.

## Nornir Configuration Files

Nornir has a few key configuration and inventory files which we need to first create.

### config.yaml

This is the file from which Nornir will get its configuration. You can also set these parameters within your Python code to overwrite the values in the _config.yaml_ file.

As you can see from the file contents, we will want to put our Python program and the _config.yaml_ file in a directory together, with a subdirectory called _inventory_ for the _hosts_, _groups_, and _defaults_ yaml files.

In my case, my _defaults.yaml_ file is empty, so I have not discussed it below.

```yaml
---
inventory:
    plugin: SimpleInventory
    options:
        host_file: "inventory/hosts.yaml"
        group_file: "inventory/groups.yaml"
        defaults_file: "inventory/defaults.yaml"
runner:
    plugin: threaded
    options:
        num_workers: 10
```

### hosts.yaml

This file is self-explanatory and contains all the hosts. In my case I am trying to define as little as possible at this level and define common attributes at the group level. As I do not have DNS running in my lab, I have defined the IP statically. By default the name of the host will be resolved and used for connectivity.

The last two hosts in the file do not exist in my lab, to help demonstrate that Nornir handles errors.

```yaml
---
xe-02:
    hostname: 192.168.217.232
    groups:
        - csr
xr-02:
    hostname: 192.168.217.251
    groups:
        - xrv
n9k-01:
    hostname: 192.168.217.250
    groups:
        - n9kv
xr-03:
    hostname: 192.168.217.2
    groups:
        - xrv
n9k-03:
    hostname: 192.168.217.25
    groups:
        - n9kv
```

### groups.yaml

Groups allow for parameters to be defined across a number of devices. Groups can be nested to allow definition of parameters at a parent level, and have inheritence throughout the nested groups.

In this case I have just adjusted the NAPALM connection timeout to 4 seconds (it is around 20 seconds by default), and have defined a few other required parameters such as username/password, and the device platform.


```yaml
---
global:
    username: admin
    password: admin
    connection_options:
        napalm:
            extras:
                timeout: 4
csr:
    platform: ios
    groups:
        - global
xrv:
    platform: iosxr_netconf
    username: root
    password: admin123
    groups:
        - global
n9kv:
    platform: nxos
    groups:
        - global
```

## Example of write to disk

The first thing I wanted to try was to write my configuration backups to disk.

In the below code I there are a few things worth mentioning:
* For IOS XE and NX-OS, NAPALM can sanitise the password hashes in the configuration file. As this is not supported for IOS XR, I have created two separate tasks to do the required backups.
* I have commented out the lines for write-to-file in Windows.
* We can look in the Python set __nr.data.failed_hosts__ to see which hosts Nornir has failed to backup.

```python
#!/usr/bin/python3

### Import required libraries
import pathlib
from nornir import InitNornir
from nornir_utils.plugins.tasks.files import write_file
from nornir_napalm.plugins.tasks import napalm_get
import requests
from requests.packages.urllib3.exceptions import InsecureRequestWarning

### Disable certificate warnings for NX-API
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

### Initialise Nornir using our configuration file
nr = InitNornir(config_file="config.yaml")

### Define a function to perform the backup
def config_backup(task):
### Configuration collection tasks
    if task.host.platform == "iosxr_netconf":
        r = task.run(
            task=napalm_get,
            getters=["config"],
            retrieve="running"
        )
    else:
        r = task.run(
            task=napalm_get,
            getters=["config"],
            retrieve="running",
            sanitized="True"
        )

### Write backup file tasks
    file = task.host
    pathlib.Path("configs").mkdir(exist_ok=True)#Linux
#    pathlib.PureWindowsPath("D:\\configs")#Windows
    task.run(
        task=write_file,
        filename=f"configs/{file}.txt",#Linux
#        filename=f"D:\\configs/{file}.txt",#Windows
        content=r.result["config"]["running"]
    )

### Import execution block
if __name__ == "__main__":
    bkp_job = nr.run(task=config_backup)

### Return all the hostnames for which the backup failed
    for each in nr.data.failed_hosts:
        print('Failed to backup host: ' + each)
```

## Example of write to GitHub SaaS repository

The real goal was to get the backups into a GitHub repository, and have them written idempotently.

To achieve this and some other improvements, a number of changes have been made in the below code. Some points worth calling out:
* As Nornir runs the function as many times as there are hosts, I have declared some variables and executed some code outside of the function. Otherwise these tasks would be repeated many times unnecessarily.
* I've written some very basic code to provide a report on the successful and failed hosts for the job.
* As with most Cisco devices there are some lines that are ever-changing. I have used __re__ to strip these out, as otherwise my configuration backup will have "changed" almost every day.
* As in reality I do want to backup the password hashes (so I know if passwords have changed), I have just performed a single backup task in the _config_backup_ function.


```python
#!/usr/bin/python3

### Import required libraries
from nornir import InitNornir
from nornir_napalm.plugins.tasks import napalm_get
import requests
from requests.packages.urllib3.exceptions import InsecureRequestWarning
import base64
from datetime import datetime
import json
import re

### Initialise Global variables
repo_slug = 'cznolan/development'
folder = 'backups'
branch = 'master'
uname = 'cznolan'
token = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
message = 'Config change detected ' + str(datetime.now().isoformat(timespec='seconds'))
path = 'https://api.github.com/repos/{}/contents/{}'.format(repo_slug, folder)
success = []

### Initialise Nornir using our configuration file
nr = InitNornir(config_file="config.yaml")

### Perform a once-off check for files in the GitHub repository folder
json_req = requests.get(path, auth=(uname, token)).json()


### Define a function to perform the backup
def config_backup(task):
### Disable certificate warnings for NX-API
### Doing this within the function as we do not
### want to disable warnings when committing to GitHub SaaS
    requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

### Configuration collection task
    r = task.run(
        task=napalm_get,
        getters=["config"],
        retrieve="running",
    )

### Defining the hosts backup so we can use it later
    task.host["backup"] = r.result["config"]["running"]

### Recording the successful hosts in a Python list
    success.append(str(task.host))


### Define a function to upload to GitHub SaaS
def github_commit(task):
    file = task.host
    file_path = '{}/{}.txt'.format(folder, file)

### Remove some dynamic configuration lines so there are not always "new" files
    if task.host.platform == "nxos":
        confbackup = str.encode(re.sub(r'(?m)^!Running configuration.*|!No configuration.*', '', task.host["backup"]))
    elif task.host.platform == "iosxr_netconf":
        confbackup = str.encode(re.sub(r'(?m)^!! Last configuration change.*\n', '', task.host["backup"]))
    else:
        confbackup = str.encode(task.host["backup"])

### Determine if the file already exists in the repository
    sha = None
    for each in json_req:
        if each['path'] == file_path:
            sha = each['sha']

### Encode the backup as base64 and prepare + upload the backup
    content = base64.b64encode(confbackup).decode()

    commitdata = {}
    commitdata["path"] = file_path
    commitdata["branch"] = branch
    commitdata["message"] = message
    commitdata["content"] = content
    if sha:
        commitdata["sha"] = str(sha)

    uploadurl = "https://api.github.com/repos/{}/contents/{}".format(repo_slug, file_path)
    requests.put(uploadurl, auth=(uname, token), data=json.dumps(commitdata))

### Import execution block
if __name__ == "__main__":
    bkp_job = nr.run(task=config_backup)
    git_job = nr.run(task=github_commit)

### Return the results
### This is verifying if the SSH backups were successful
### Does not quite verify if the upload to GitHub SaaS was successful
    if bool(nr.data.failed_hosts) is False:
        print('Successfully backed up all hosts: ' + str(success))
    else:
        print('Successfully backed up hosts: ' + str(success))
        print('Failed to backup hosts: ' + str(nr.data.failed_hosts))

```

## Results

The resulting output is that we get a backup file for each host inside the GitHub SaaS repository _backups_ folder.

![Backup Files](/images/2021-06-13-nornir-repo.PNG)

After an initial backup and then configuring the regexes to remove the ever-changing configuration lines, we can see that the file has been re-committed with the unwanted line removed.

![Line Removed](/images/2021-06-13-nornir-re.PNG)

## Additional Links

Take a look at the connectivity options available for NAPALM, as you may want/need to enable NETCONF or the NX-API on your devices.

[https://napalm.readthedocs.io/en/stable/support/](https://napalm.readthedocs.io/en/stable/support/){:target="_blank"}

All plugins have been removed from the Nornir core application package, so you may wish to take a look here for a few plugins that are available as additional downloads.

[https://nornir.tech/nornir/plugins/](https://nornir.tech/nornir/plugins/){:target="_blank"}
