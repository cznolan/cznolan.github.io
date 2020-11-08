---
layout: post
title: "Cisco SSM On-Prem - XE, XR, NX-OS Configuration"
date: 2020-11-08 11:00:00 +0800
tags: cisco ios xe xr nx-os ssm smart licence
author: cznolan
---
Recently I have deployed a Cisco Smart Software Manager (SSM) On-Prem server for a customer in order for them to use Smart Licencing on their IOS XE, IOS XR, and NX-OS devices.

The setup of SSM On-Prem is relatively easy to fumble your way through. However, when it came to configuring each type of device to use Smart Licencing there was little consistency in configuration commands and behaviour. The intention of this article is to provide working configuration for Cisco network devices in order to Smart Licence them through an SSM On-Prem instance.

## Smart Software Manager On-Prem

My couple of key callouts around SSM On-Prem configuration are as follows
* The SSM On-Prem certificate Common Name and Subject Alternative Name FQDN must be resolvable _using DNS protocol_ by the CSSM On-Prem server. You cannot setup a static host entry for these FQDNs on the SSM On-Prem server. If you cannot make these resolvable using DNS, you may use an IP address for these values.
* Network devices should use the certificate Common Name (be it FQDN or IP) in their Smart Licence configuration, as some products have strict certificate checking. For some products such as IOS XE this can be disabled using the __no http secure server-identity-check__ configuration command.
* Ensure that the network device licences are in the correct Virtual Account that the SSM On-Prem server is registered to, and that the ID Token has been issued on the SSM On-Prem server and not in the SSM cloud portal.
* Some products do not work with SSM On-Prem, so check the documentation. Identity Services Engine can only use direct SSM cloud connectivity or the Transport Gateway. Application Centric Infrastructure can only do Device Led Conversion directly to the SSM cloud, but can use SSM On-Prem for ongoing licencing after the initial DLC.

## IOS XE Devices

For devices running IOS XE, there are two approaches for Smart Licencing - the older Smart Call Home method, and the newer and recommended Smart Transport method. I have tested on IOS XE 16.12.4 and 17.3.1a.

### Smart Transport

With Smart Transport being the recommended option from Cisco and the simplest to setup, I also recommend that you use it as your smart licence communication mechanism.

The below example assumes you wish to use the Gi0/0 management interface in the Mgmt-vrf VRF.

```
ip name-server vrf Mgmt-vrf 10.10.10.10

license smart url https://ssm-onprem.example/SmartTransport
license smart transport smart

ip http client source-interface GigabitEthernet0/0

crypto pki trustpoint SLA-TrustPoint
 revocation-check crl none
```

### Smart Call Home

Smart Call Home is a bit more finicky than Smart Transport. If you are using a named VRF for the transport, such as the device management interface, Call Home attempts to make the DNS lookup in the global VRF. You will have to have DNS functioning in the global VRF, or put a static host entry _in the global VRF_ for Smart Call Home to work within a named VRF. Putting a static host entry in the named VRF did not work for me, but it may be worth trying that in future.

You can also use a named Call Home profile rather than the default Cisco-TAC1 profile if you wish.

The below example assumes you wish to use the Gi0/0 management interface in the Mgmt-vrf VRF.

```
license smart transport callhome

ip http client source-interface GigabitEthernet0/0

ip host ssm-onprem.example 10.20.20.20

crypto pki trustpoint SLA-TrustPoint
 revocation-check crl none

call-home
 vrf Mgmt-vrf
 profile "cssm-onprem"
  reporting smart-licensing-data
  destination transport-method http
  destination address http https://ssm-onprem.example/Transportgateway/services/DeviceRequestHandler
  active
```

## IOS XR

At time of writing the latest IOS XR version is 7.1, and the only option for Smart Licencing is Smart Call Home. Smart Transport has not been introduced to the IOS XR platforms as yet. I have tested on IOS XR 7.0.2.

### Smart Call Home

The key thing to note with IOS XR is that the hidden command __http client secure-verify-peer disable__ is required to be configured. This command is hidden within the configuration, as well as being hidden from CLI command completion when using Tab/? at the configuration prompt.

The below example assumes you wish to use the MgmtEth0/RSP0/CPU0/0 management interface in the management VRF.

```
domain vrf management name-server 10.10.10.10

crypto ca trustpoint Trustpool
 crl optional

http client secure-verify-peer disable

call-home
 vrf management
 service active
 contact smart-licensing
 source-interface MgmtEth0/RSP0/CPU0/0
 profile cssm-onprem
  active
  destination address http https://ssm-onprem.example/Transportgateway/services/DeviceRequestHandler
  reporting smart-call-home-data
  reporting smart-licensing-data
  destination transport-method http
```

## NX-OS

At time of writing the latest NX-OS version is 9.3, and the only option for Smart Licencing is Smart Call Home. Smart Transport has not been introduced to the NX-OS platforms as yet. I have tested on NX-OS 9.3.5.

### Smart Call Home

On NX-OS the Call Home process will attempt to do the Smart Licencing using the default Cisco-TAC1 profile, so you must update that profile with your appropriate Smart Licencing configuration.

The below example assumes you wish to use the mgmt0 management interface in the management VRF.

```
license smart enable

ip http source-interface mgmt0 vrf management

vrf context management
 ip name-server 10.10.10.10

callhome
 destination-profile CiscoTAC-1 transport-method http
 destination-profile CiscoTAC-1 index 1 http https://ssm-onprem.example/Transportgateway/services/DeviceRequestHandler
 transport http use-vrf management
 enable
```

## Useful Commands

Below are some additional useful/necessary commands that are used in Smart Licencing.

To register the device to the SSM On-Prem instance use the `license smart register idtoken <token>` exec command.

To convert a traditional licence on a device to a Smart Licence, use the `license smart conversion start` exec command. This process is referred to as Device Led Conversion (DLC).

To refresh the authorisation of the device to the SSM On-Prem instance, as well as refresh any consumed licences, use the `license smart renew auth` exec command.

For debugging, I found the `debug call-home all` command to be the most useful, however it obviously only works when using Call Home.
