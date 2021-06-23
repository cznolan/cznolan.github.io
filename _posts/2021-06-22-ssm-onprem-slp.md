---
layout: post
title: "Cisco SSM On-Prem - IOS XE Smart Licencing Using Policy (SLP)"
date: 2021-06-22 20:30:00 +0800
tags: cisco ios xe cssm ssm smart licence using policy slp
author: cznolan
---
In a previous blog post I explained how to setup IOS XE, IOS XR, and NX-OS for Smart Licencing using SSM On-Prem, and explained both the Smart Transport and Smart Call Home methods with IOS XE. Finally, Cisco have released a version of SSM On-Prem that supports the IOS XE 17.3.2 and onwards licencing method of Smart Licencing Using Policy (SLP).

This post is just a quick update on how to configure IOS XE devices for SLP using SSM On-Prem.

For other devices and earlier IOS XE versions, take a look at my previous blog post.

[https://cznolan.github.io/2020/11/08/ssm-onprem-deviceconfig.html](https://cznolan.github.io/2020/11/08/ssm-onprem-deviceconfig.html){:target="_blank"}

## Smart Software Manager On-Prem

You will need at least SSM On-Prem version 8-202102 to support SLP, which was released in May 2021 (contrary to the Feb 2021 build date). I would however recommend using version 8-202105 as the earlier build introduced a bug affecting Smart Licencing for Prime Infrastructure and UC applications.

## Smart Licencing using Policy (SLP)

SLP is configured similar to both the Smart Transport and Cisco Smart Licencing Utility (CSLU) methods. In the SSM On-Prem web interface, if you navigate to _Smart Licencing > Inventory > General_, you will find some text under the __Product Instance Registration Tokens__ heading which will include a link for the __CSLU Transport URL__. Presumably, you will find the name of the virtual account being managed by SSM On-Prem must be appended to the standard CSLU URI along with _-1_. So, if your virtual account is called _myaccount_, your URL would look something like the following.

_https://ssm-onprem.example/cslu/v1/pi/myaccount-1_

```
ip name-server vrf Mgmt-vrf 10.10.10.10

license smart url https://ssm-onprem.example/cslu/v1/pi/<account>-1
license smart transport cslu

ip http client source-interface GigabitEthernet0/0

crypto pki trustpoint SLA-TrustPoint
 revocation-check crl none
```

## Troubleshooting

If you have not yet upgraded SSM On-Prem to a version supporting SLP, you will see the following syslog message on devices running IOS XE 17.3.2 and above.

_%SMART_LIC-4-REPORTING_NOT_SUPPORTED: Smart Agent for Licensing CSSM OnPrem is down rev and does not support the enhanced policy and usage reporting mode._

If you have misconfigured the devices, such as still having Smart Transport configured on a device running an IOS XE version requiring SLP, you are likely to see the following syslog message.

_%SMART_LIC-3-COMM_FAILED: Communications failure with the Cisco Smart Software Manager (CSSM) : Received empty response from server_

Once everything is configured correctly, you should see the following syslog message.

_%SMART_LIC-5-COMM_RESTORED: Communications with Cisco Smart License Utility (CSLU) restored_

Additionally, if your CRL check is failing you are likely to see periodic syslog messages reporting this. You can disable the CRL check entirely by configuring __revocation-check none__ to avoid these messages as desired.
