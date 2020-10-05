---
layout: post
title: "Basic Cisco ACI API Calls With Python"
date: 2020-10-01 20:45:00 +0800
tags: cisco aci apic python rest api
author: cznolan
---
This post is a getting started guide on making REST API calls to the Cisco ACI APIC using Python 3. I have intended this to be targeted at people who have some basic understanding of both Python and Cisco ACI, but don't really know how to get started with making REST API calls to the APIC.

I have deliberately unoptimised the Python code so that it is clear what is actually happening, which will hopefully help you to adapt it to your purposes. The first five examples are simple GET operations, and the final example is a POST operation where changes are actually made to the ACI configuration.

I would recommend that you have some understanding of:
* How to use Python modules.
* How to work with Python lists and dictionaries, as JSON arrays and objects are treated as such by Python.
* Cisco ACI constructs, predominately so that you have some idea of what information is useful.

## Basic Setup

In these examples I am using Python 3.8 on Windows 10, and interacting with Cisco's always-on APIC Sandbox which is running version 4.1(1k). I often use similar Python scripts on APICs running 3.2 or 4.2.

The REST API can return data in JSON or XML format. Due to personal preference this guide will only deal with JSON.

### Python Modules

At a minimum you will need the __json__ and __requests__ modules. I also recommend using __pprint__ to neatly observe the JSON output, as well as disabling validation of the APIC certificate.
```python
import json
import requests
import pprint
from requests.packages.urllib3.exceptions import InsecureRequestWarning

requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
```

### Authentication

I see no reason to re-invent the wheel here. Cisco has a clear and concise piece of Python code here that tells you exactly how to interact with the APIC.

[https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/1-x/api/rest/b_APIC_RESTful_API_User_Guide/using_tools_for_api_development_and_testing.pdf](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/1-x/api/rest/b_APIC_RESTful_API_User_Guide/using_tools_for_api_development_and_testing.pdf){:target="_blank"}

The only thing I will do above what is in the PDF is define the username and passwords as variables. I have also sanitised the APIC Sandbox FQDN and credentials, as while they are readily available through a Google search today, I am not sure if Cisco will restrict it in future.

For the purpose of authentication we just need the following Python code. This will provide us with a token that we can present in future API calls. The token will likely expire after a short period so if you are running your API calls 5 or more minutes apart you may wish to authenticate every time.

```python
# basic parameters
base_url = 'https://192.0.2.1/api/'
apicUser = 'admin'
apicPassword = 'admin'

# create credentials structure
name_pwd = {'aaaUser': {'attributes': {'name': apicUser, 'pwd': apicPassword}}}
json_credentials = json.dumps(name_pwd)

# log in to API
login_url = base_url + 'aaaLogin.json'
post_response = requests.post(login_url, data=json_credentials, verify=False)

# get token from login response structure
auth = json.loads(post_response.text)
login_attributes = auth['imdata'][0]['aaaLogin']['attributes']
auth_token = login_attributes['token']

# create cookie array from token
cookies = {}
cookies['APIC-Cookie'] = auth_token
```

## Examples

The following are some basic examples of read and write operations against the APIC REST API. The example code should be combined with the above module import and authentication sections.

If you want to see the raw JSON output returned from the HTTP GET request (using __requests.get__), I recommend using __pprint__ as follows.

```python
pprint.pprint(get_response.json())
```

Example output:

```json
{'imdata': [{'fvTenant': {'attributes': {'annotation': '',
                                         'childAction': '',
                                         'descr': '',
                                         'dn': 'uni/tn-infra',
                                         'extMngdBy': '',
                                         'lcOwn': 'local',
                                         'modTs': '2020-10-05T01:04:27.379+00:00',
                                         'monPolDn': 'uni/tn-common/monepg-default',
                                         'name': 'infra',
                                         'nameAlias': '',
                                         'ownerKey': '',
                                         'ownerTag': '',
                                         'status': '',
                                         'uid': '0'}}},
            {'fvTenant': {'attributes': {'annotation': '',
                                         'childAction': '',
                                         'descr': '',
                                         'dn': 'uni/tn-mgmt',
                                         'extMngdBy': '',
                                         'lcOwn': 'local',
                                         'modTs': '2020-10-05T01:04:27.572+00:00',
                                         'monPolDn': 'uni/tn-common/monepg-default',
                                         'name': 'mgmt',
                                         'nameAlias': '',
                                         'ownerKey': '',
                                         'ownerTag': '',
                                         'status': '',
                                         'uid': '0'}}},
            {'fvTenant': {'attributes': {'annotation': '',
                                         'childAction': '',
                                         'descr': '',
                                         'dn': 'uni/tn-common',
                                         'extMngdBy': '',
                                         'lcOwn': 'local',
                                         'modTs': '2020-10-05T01:04:20.182+00:00',
                                         'monPolDn': 'uni/tn-common/monepg-default',
                                         'name': 'common',
                                         'nameAlias': '',
                                         'ownerKey': '',
                                         'ownerTag': '',
                                         'status': '',
                                         'uid': '0'}}},
            {'fvTenant': {'attributes': {'annotation': '',
                                         'childAction': '',
                                         'descr': '',
                                         'dn': 'uni/tn-Heroes',
                                         'extMngdBy': '',
                                         'lcOwn': 'local',
                                         'modTs': '2020-10-05T01:07:01.652+00:00',
                                         'monPolDn': 'uni/tn-common/monepg-default',
                                         'name': 'Heroes',
                                         'nameAlias': '',
                                         'ownerKey': '',
                                         'ownerTag': '',
                                         'status': '',
                                         'uid': '15374'}}},
            {'fvTenant': {'attributes': {'annotation': '',
                                         'childAction': '',
                                         'descr': '',
                                         'dn': 'uni/tn-SnV',
                                         'extMngdBy': '',
                                         'lcOwn': 'local',
                                         'modTs': '2020-10-05T01:07:01.967+00:00',
                                         'monPolDn': 'uni/tn-common/monepg-default',
                                         'name': 'SnV',
                                         'nameAlias': '',
                                         'ownerKey': '',
                                         'ownerTag': '',
                                         'status': '',
                                         'uid': '15374'}}},
 'totalCount': '5'}
 ```

### List all tenants

This will query the APIC for all the tenants, and will return the name of them all in a simple text list. The number of tenants is also derived from the __totalCount__ object.

```python
# query tenants
query_url = base_url + 'class/fvTenant.json'
get_response = requests.get(query_url, cookies=cookies, verify=False)

# print total tenant count and name of tenants
result_len = get_response.json().get('totalCount')
print('Number of results: {}'.format(result_len))

for each in get_response.json()['imdata']:
    print(each['fvTenant']['attributes'].get('name'))
```

Example output:
```
Number of results: 5
infra
mgmt
common
Heroes
SnV
```

### Inventory of fabric switches

This will query the APIC for all the fabric nodes such as leaf and spine switches, as well as the APICs. The returned output is then printed out as comma delimited data.

```python
# query all fabric nodes
query_url = base_url + 'node/class/fabricNode.json'

# perform the query
get_response = requests.get(query_url, cookies=cookies, verify=False)

# print useful information from each node
result_len = get_response.json().get('totalCount')
print('Number of fabric nodes: {}'.format(result_len))

print('node_id,model,name,role,serial')
for each in get_response.json()['imdata']:
    node_id = each['fabricNode']['attributes'].get('id')
    node_model = each['fabricNode']['attributes'].get('model')
    node_name = each['fabricNode']['attributes'].get('name')
    node_role = each['fabricNode']['attributes'].get('role')
    node_serial = each['fabricNode']['attributes'].get('serial')
    print('{},{},{},{},{}'.format(node_id, node_model, node_name, node_role, node_serial))
```

Example output:
```
Number of fabric nodes: 4
node_id,model,name,role,serial
102,N9K-C9396PX,leaf-2,leaf,12345678
201,N9K-C9508,spine-1,spine,87654321
1,APIC-SERVER-L2,apic1,controller,56781234
101,N9K-C9396PX,leaf-1,leaf,13572468
```

### List of bridge domains and attributes

This will query the APIC for all the bridge domains within a given tenant, and provide some basic configuration parameters of each bridge domain. Try & Except have been used to handle any errors that occur if a bridge domain does not have any defined subnets.

```python
# additional parameters
tenant = 'tenant_name'

# direct the query at a specific tenant
query_url = base_url + 'node/mo/uni/tn-{}.json?query-target=children&target-subtree-class=fvBD&rsp-subtree=full&rsp-subtree-class=fvSubnet'.format(tenant)

# perform the query
get_response = requests.get(query_url, cookies=cookies, verify=False)

# return information from each BD in tenant
result_len = get_response.json().get('totalCount')
print('Number of results: {}'.format(result_len))
print('------------------------------------')
for each in get_response.json()['imdata']:
    print('BD Name: ' + each['fvBD']['attributes'].get('name'))
    print('ARP Flooding: ' + each['fvBD']['attributes'].get('arpFlood'))
    print('Limit IP Learning to Subnets: ' + each['fvBD']['attributes'].get('limitIpLearnToSubnets'))
    print('Gatway MAC: ' + each['fvBD']['attributes'].get('mac'))
    print('Multi-Destination Packet Action: ' + each['fvBD']['attributes'].get('multiDstPktAct'))
    print('Unicast Routing: ' + each['fvBD']['attributes'].get('unicastRoute'))
    print('Unknown Multicast Action: ' + each['fvBD']['attributes'].get('unkMcastAct'))
    try:
        for each in each['fvBD']['children']:
            print('Subnet: ' + each['fvSubnet']['attributes'].get('ip'))
    except:
        pass
    print('------------------------------------')
```

Example output:
```
Number of results: 2
------------------------------------
BD Name: CLIENT01_BD
ARP Flooding: no
Limit IP Learning to Subnets: yes
Gatway MAC: 00:22:BD:F8:19:FF
Multi-Destination Packet Action: bd-flood
Unicast Routing: yes
Unknown Multicast Action: flood
------------------------------------
BD Name: Hero_Land
ARP Flooding: no
Limit IP Learning to Subnets: yes
Gatway MAC: 00:22:BD:F8:19:FF
Multi-Destination Packet Action: bd-flood
Unicast Routing: yes
Unknown Multicast Action: flood
Subnet: 192.168.120.1/22
Subnet: 10.1.120.1/22
------------------------------------
```

### List all configured EPGs on an interface

This will query the APIC for a specific interface and list out the names, encapsulation type, and VLAN tag for EPGs on that interface. The information is presented in a comma delimited format.
Examples are included for both directing this query at a standalone interface as well as at a vPC interface. You should comment out whichever line is not required.

```python
# additional parameters
pod_id = '1'
leaf_id = '101'
leaf_int = 'eth1/1'
vpc_pair = '101-102'
vpc_int = 'vpc_name'

# direct the query at a standalone interface
query_url = base_url + 'class/fvRsPathAtt.json?query-target-filter=eq(fvRsPathAtt.tDn,"topology/pod-{}/paths-{}/pathep-[{}]")'.format(pod_id, leaf_id, leaf_int)

# direct the query at a vpc interface
query_url = base_url + 'class/fvRsPathAtt.json?query-target-filter=eq(fvRsPathAtt.tDn,"topology/pod-{}/protpaths-{}/pathep-[{}]")'.format(pod_id, vpc_pair, vpc_int)

# perform the query
get_response = requests.get(query_url, cookies=cookies, verify=False)

# print total epg count and name of epgs
result_len = get_response.json().get('totalCount')
print('Number of results: {}'.format(result_len))
for each in get_response.json()['imdata']:
    epg_name = each['fvRsPathAtt']['attributes'].get('dn').split('/')[3].lstrip('epg-')
    epg_encap = each['fvRsPathAtt']['attributes'].get('encap')
    epg_mode = each['fvRsPathAtt']['attributes'].get('mode')
    print(epg_name + ',' + epg_encap + ',' + epg_mode)
```

Example output:
```
Number of results: 3
app,vlan-201,regular
db,vlan-202,regular
web,vlan-200,regular
```
### List all endpoints in a given EPG

This will query the APIC for all the MAC addresses (endpoints) in a given EPG, and provide the corresponding IPv4 and IPv6 addresses of each endpoint.

```python
# additional parameters
tenant = 'tenant_name'
app_profile = 'application_profile'
epg_id = 'epg_name'

# direct the query at a specific EPG
query_url = base_url + 'node/mo/uni/tn-{}/ap-{}/epg-{}.json?query-target=children&target-subtree-class=fvCEp&rsp-subtree=children&rsp-subtree-class=fvIp'.format(tenant, app_profile, epg_id)

# perform the query
get_response = requests.get(query_url, cookies=cookies, verify=False)

# return number of MACs found in EPG and all IP addresses known for each MAC
result_len = get_response.json().get('totalCount')
print('Number of MAC addresses: {}'.format(result_len))
print('------------------------------------')
for each in get_response.json()['imdata']:
    print('MAC: ' + each['fvCEp']['attributes'].get('mac'))
    for each in each['fvCEp']['children']:
        print('IP: ' + each['fvIp']['attributes'].get('addr'))
    print('------------------------------------')
```

Example output:
```
Number of MAC addresses: 2
------------------------------------
MAC: 46:CD:BB:C0:00:00
IP: 2222::66:4
IP: 10.193.102.4
------------------------------------
MAC: 43:CD:BB:C0:00:00
IP: 2222::65:2
IP: 10.193.101.2
IP: 10.193.102.1
IP: 2222::66:1
------------------------------------
```

### Add an EPG to an interface (static binding)

This will create a static binding for an EPG on the given interface.
Examples are included for both directing this query at a standalone interface as well as at a vPC interface. You should comment out whichever lines are not required.

```python
# additional parameters
tenant = 'tenant_name'
app_profile = 'application_profile'
epg_id = 'epg_name'
epg_encap = 'vlan-201'

# define post url
query_url = base_url + 'node/mo/uni/tn-{}/ap-{}/epg-{}.json'.format(tenant, app_profile, epg_id)

# apply the EPG to a standalone interface
epg_data = {'fvRsPathAtt': {'attributes': {'encap': epg_encap, 'mode': 'regular', 'tDn': 'topology/pod-1/paths-101/pathep-[eth1/37]', 'status': 'created'},'children':[]}}

# apply the EPG to a vpc interface
epg_data = {'fvRsPathAtt': {'attributes': {'encap': epg_encap, 'mode': 'regular', 'tDn': 'topology/pod-1/protpaths-101-102/pathep-[SnV_FI-2A]', 'status': 'created'},'children':[]}}

# apply the EPG and confirm response
data_payload = json.dumps(epg_data)
post_response = requests.post(query_url, data=data_payload, cookies=cookies, verify=False)
print(post_response)
```

Example output:

You will simply get a HTTP 200 response if the operation is valid.

```
<Response [200]>
```

## Resources

I recommend you take a look at the below resources when trying to work with the APIC REST API.

### APIC REST API Configuration Guide

To both optimise your queries and ensure that the data you are looking for is returned, it is good to understand the query filtering/scoping expressions that can be appended to the query URL. Examples and descriptions of the query filters can be found in the configuration guide.

[https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/4-x/rest-api-config/Cisco-APIC-REST-API-Configuration-Guide-42x/m_using_the_rest_api.html](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/4-x/rest-api-config/Cisco-APIC-REST-API-Configuration-Guide-42x/m_using_the_rest_api.html){:target="_blank"}

### APIC Object Store Browser

The Object Store Browser allows you to explore the APIC Management Information Tree (MIT) and browse through all the objects that you can interact with using the REST API. The tool is accessed by simply appending _/visore.html_ to the APIC URL, for example https://apic/visore.html

You can also get to Visore by clicking on the cog wheel at the top-right corner of the APIC web GUI, and selecting "Object Store Browser".

### APIC API Explorer

The API Explorer shows you what API calls are being made to provide the information displayed in the APIC web GUI. This is really useful if you know how to find something in the web GUI, but don't know where to start when trying to get the same data using the REST API.

To access the API Explorer, click on the cog wheel at the top-right corner of the APIC web GUI, and select "Show API Explorer".

### Cisco DevNet

Cisco DevNet has lots of great information and resources, including free Sandbox environments, follow-along labs, and links to GitHub repos with code examples.

[https://developer.cisco.com/](https://developer.cisco.com/){:target="_blank"}
