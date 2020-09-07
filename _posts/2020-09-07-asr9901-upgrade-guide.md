---
layout: post
title: "Cisco ASR9901 Upgrade Guide"
date: 2020-09-07 20:45:00 +0800
tag: IOS XR
author: cznolan
---
I recently deployed several Cisco ASR9901 routers and have realised that a lot of the information on IOS XR platforms is in limbo between the older 32-bit QNX-based version of the operating system, and the newer 64-bit Wind River Linux-based version. I thought it might be helpful to publish an up-to-date IOS XR upgrade guide to help people who are navigating IOS XR for the first time.

This post is a step-by-step walkthrough of upgrading the ASR9901, and the procedure has been tested on IOS XR upgrades from 6.5.2 > 6.6.3 as well as 6.6.3 > 7.0.2.

## Procedure

### Files Required

First thing first, once you have chosen which version of code you want to go with, you need to download it. There are two files you are going to need:
1. The base IOS XR operating system, which is sometimes referred to as the "mini" or Minimum Boot Image.
1. The PX bundle. On IOS XR 64-bit these are Red Hat Package Manager (RPM) files, but documentation may make reference to Package Installation Envelopes (PIE) which was the correct nomenclature in IOS XR 32-bit. Don't ask me what PX stands for, but I assume it is a reference to the Route Processor's…processor (P) and the x86 instruction set (X).

In the case of IOS XR 7.0.2 on the ASR9901 the two files you need would be:

    asr9k-mini-x64-7.0.2.iso
    ASR9K-x64-iosxr-px-k9-7.0.2.tar

If you are so inclined, this would be the time to copy the MD5 hash of the files so you can verify them before installation.

### Pre-Upgrade Tasks

The next thing you will want to do are some pre-upgrade checks.

    show install active
    show install commit
    show alarms
    admin show platform

**show install active** will show you what packages are activated (currently running) on the system. You are going to want to make sure that you install the updated versions of all active packages as part of your upgrade.

    RP/0/RSP0/CPU0:XR1#show install active
    Node 0/RSP0/CPU0 [RP]
      Boot Partition: xr_lv0
      Active Packages: 2
            asr9k-xr-6.6.3 version=6.6.3 [Boot image]
            asr9k-k9sec-x64-2.1.0.0-r663
    
    Node 0/0/CPU0 [LC]
      Boot Partition: xr_lv0
      Active Packages: 2
            asr9k-xr-6.6.3 version=6.6.3 [Boot image]
            asr9k-k9sec-x64-2.1.0.0-r663

**show install commit** will show you what packages have been committed. You will want to make sure this output is the same as your **show install active** output, and if it isn't I would suggest doing an **install commit** before going any further with the upgrade to commit the currently active software release and packages.

**show alarms** will show you any outstanding alarm states on the system. There may be some alarms like outdated FPDs which would be worth resolving before proceeding with the upgrade.

**admin show platform** is a quick and easy way to check that your hardware and software is in a healthy state.

    RP/0/RSP0/CPU0:XR1#admin show platform
    Location  Card Type               HW State      SW State      Config State  
    ----------------------------------------------------------------------------
    0/0       ASR-9901-LC             OPERATIONAL   OPERATIONAL   NSHUT         
    0/RSP0    ASR-9901-RP             OPERATIONAL   OPERATIONAL   NSHUT         
    0/FT0     ASR-9901-FAN            OPERATIONAL   N/A           NSHUT         
    0/FT1     ASR-9901-FAN            OPERATIONAL   N/A           NSHUT         
    0/FT2     ASR-9901-FAN            OPERATIONAL   N/A           NSHUT         
    0/PT0     A9K-AC-PEM              OPERATIONAL   N/A           NSHUT         


If you want to proceed with the upgrade, then the first thing would be to copy the files we downloaded from the Cisco website onto the system.

In my case I am connecting to the FTP server using the management interface which is configured in the *management* VRF

    copy ftp://192.0.2.1/asr9k-mini-x64-7.0.2.iso disk0: vrf management
    copy ftp://192.0.2.1/ASR9K-x64-iosxr-px-k9-7.0.2.tar disk0: vrf management

Once you have copied the files onto the system, if you did the copy operation with TFTP you may wish to verify the MD5 hash of the files against what is published on the Cisco website. To do this you can simply run the following commands:

    show md5 file /disk0:/asr9k-mini-x64-7.0.2.iso
    show md5 file /disk0:/ASR9K-x64-iosxr-px-k9-7.0.2.tar

And you will get output similar to below:

    RP/0/RSP0/CPU0:XR1#show md5 file /disk0:/asr9k-mini-x64-7.0.2.iso
    686f388ea27b276c506d1dfb185087f7

### Software Preparation

If the MD5 hashes match, then the next step is to add the files that we uploaded to the router into the software source repository. This is done using the following commands.

    install add source /disk0: asr9k-mini-x64-7.0.2.iso
    install add source /disk0: ASR9K-x64-iosxr-px-k9-7.0.2.tar


Note that a lot of operations in IOS XR can occur asynchronously (in the background), so you can do other things while you wait for the software to be added to the repository. As you can see from the below output this process only took about 90 seconds. If you are desperate to check on the progress you can use the show install request command to see the progress.

    RP/0/RSP0/CPU0:XR1#install add source /disk0: asr9k-mini-x64-7.0.2.iso
    
    Thu May 14 04:01:03.381 UTC
    May 14 04:01:05 Install operation 7 started by netopsroot:
     install add source /disk0: asr9k-mini-x64-7.0.2.iso 
    May 14 04:01:13 Install operation will continue in the background
    RP/0/RSP0/CPU0:XR1#
    RP/0/RSP0/CPU0:XR1#May 14 04:02:34 Install operation 7 finished successfully

You can then check the repository and take a note of which packages you need to install. If you are not sure exactly which packages you might need it is best to refer to the release notes for the specific release on which features/functionality are included in each package. At a minimum you are going to need whatever you currently have installed based on the output of the **show install active** command, and based on the earlier output of this command we just need the two packages shown below.

    RP/0/RSP0/CPU0:XR1#show install repository
    29 package(s) in XR repository:
        asr9k-mini-x64-7.0.2
        asr9k-k9sec-x64-2.1.0.0-r702.x86_64
        ...etc

Once you have decided on the packages you want to install, I recommend using the install prepare command to prepare the packages for activation. The prepare operation both ensures that the packages are not corrupted, and also reduces the duration of time the router is unusable when you ultimately activate the packages.

The install prepare command accepts multiple arguments so you can prepare all required packages for installation with a single command.

    install prepare asr9k-mini-x64-7.0.2 asr9k-k9sec-x64-2.1.0.0-r702.x86_64

The install prepare process goes through three phases of preparation against the Host, SysAdmin, and XR platform components. The progress of each of these stages can be monitored using the **show install request** command.

    RP/0/RSP0/CPU0:XR1#show install request
    
    Thu May 14 04:58:02.775 UTC
    User root, Op Id 13
    install prepare  
    asr9k-mini-x64-7.0.2
    asr9k-k9sec-x64-2.1.0.0-r702.x86_64
    
    The install prepare operation 13 is 40% complete
    
     install prepare operation 13 is in progress
    Host preparation is complete
    Sysadmin preparation is complete
    XR preparation is in progress
    
    Node(s)                 State                   %Completed              Action
    --------------------------------------------------------------------------------
    Node 0/RSP0     :       Completed                  100%                 XR prepare is complete
    Node 0/0        :       In Progress                30%                  XR prepare in progress


Once the router has signalled that the prepare process is complete, you can verify using the **show install prepare** command. If you are only installing a package or SMU, this will also let you know if a reboot is required.

    RP/0/RSP0/CPU0:XR1#show install prepare
    Prepared Boot Image:  asr9k-mini-x64-7.0.2
    Prepared Boot Partition:  /dev/panini_vol_grp/xr_lv12
    Restart Type: Reboot
    Prepared Packages:
     asr9k-mini-x64-7.0.2
     asr9k-k9sec-x64-2.1.0.0-r702

### Software Upgrade

The next step is to activate the new software version, which requires a system reload, which is obviously disruptive. If you want to back out now use the *install prepare clean* command to back out the earlier install prepare operation.

Should you wish to proceed, issue the **install activate** command to activate the software and reload the system. You can monitor the progress using the *show install request* command up until the system reboots. The reboot should not take more than 5-10 minutes.

    install activate
	
Once your system is back online, you can verify you are running the new version using the *show version*, but more importantly you will want to run the **show install active** command to verify the IOS XR version along with active packages.

Once you have performed your verification steps of the new software version, you need to commit the new software and packages, or you are going to undo all your hard work very shortly when you reload again. You can leave the router running for an extended period with an un-commited software upgrade, but should it reboot for any reason the software will roll back.

    install commit

### FPD Upgrade

Now that you have *committed* to the new software and package versions, you need to verify that the FPDs are all running a compatible version of software.

    show hw-module fpd

If any modules are in the "NEED UPGD" Status, use the following command to upgrade all modules requiring upgrade.

    upgrade hw-module location all fpd all

Progress can be monitored using the below command:

    show hw-module fpd

Once all modules are in the CURRENT or RLOAD REQ Status, reload the HW modules to finish the module upgrade. Note that this is effectively reloading the system and will roll back your software upgrade if you have not yet performed the *install commit*.

    hw-module location <location> reload
	
*NOTE: The upgrade and reload may need to be done from SysAdmin VM.*

    RP/0/RSP0/CPU0:XR1#admin
    Thu May 14 05:18:19.242 UTC
    root connected from 127.0.0.1 using console on sysadmin-vm:0_RSP0
    sysadmin-vm:0_RSP0# upgrade hw-module location all fpd all
    Thu May  14 05:18:32.985 UTC+00:00
    sysadmin-vm:0_RSP0# 
    sysadmin-vm:0_RSP0# hw-module location 0/0 reload 
    Thu May  14 05:20:16.203 UTC+00:00
    Reload hardware module ? [no,yes] yes
    result Card graceful reload request on 0/0 succeeded.
    sysadmin-vm:0_RSP0#

The module restart process will take at least 5 minutes. It may also take some time for the FPDs to become ready after the module has rebooted. Monitor the status of the FPDs using the **show hw-module fpd** command.

Verify the modules are all CURRENT and there are no package issues using the below commands:

    show hw-module fpd
    show alarms brief

If you find that the **show hw-module fpd** command shows the modules as CURRENT, but the **show alarms brief** command shows that the FPDs are below the minimum version, try upgrading the FPDs again using the *force* keyword.

    RP/0/RSP0/CPU0:XR1#show alarms brief
    --------------------------------------------------------------------------------
    System Scoped Active Alarms 
    --------------------------------------------------------------------------------
    Location        Severity     Group            Set Time                   Description                                                                                                                                                                                                                                                
    --------------------------------------------------------------------------------
    0/RSP0          Major        FPD_Infra        07/15/2020 11:07:44 AWST   IPU-FPGA: Golden FPGA is below minimum version, Perform force fpd upgrade                                                                                                                                                                                  
    0/RSP0          Major        FPD_Infra        07/15/2020 11:07:44 AWST   IPU-FSBL: Golden FPGA is below minimum version, Perform force fpd upgrade                                                                                                                                                                                  
    0/RSP0          Major        FPD_Infra        07/15/2020 11:07:44 AWST   IPU-Linux: Golden FPGA is below minimum version, Perform force fpd upgrade                                                                                                                                                                                 
    0/0             Major        FPD_Infra        02/06/2020 15:19:51 AWST   IPU-FPGA: Golden FPGA is below minimum version, Perform force fpd upgrade                                                                                                                                                                                  
    0/0             Major        FPD_Infra        02/06/2020 15:19:51 AWST   IPU-FSBL: Golden FPGA is below minimum version, Perform force fpd upgrade                                                                                                                                                                                  
    0/0             Major        FPD_Infra        02/06/2020 15:19:51 AWST   IPU-Linux: Golden FPGA is below minimum version, Perform force fpd upgrade

### Cleanup

If no longer required, remove any packages that are no longer required. The install remove command accepts multiple arguments so you can remove multiple packages at once.

    show install inactive

    install remove asr9k-xr-6.6.3 asr9k-k9sec-x64-2.1.0.0-r663.x86_64

### Troubleshooting

Troubleshooting steps will vary depending on where the install process fails. Good things to check are:
* File/package names are correct. Make sure you aren't typing a file name when you should be typing a package name and vice versa.
* Make sure your install prepare/activate command calls all currently-activated packages. Don't try to exclude activated packages during the upgrade process.
* Try to do as much verification as you can before you do your *install commit*, as it is easy to just reload the router to undo your software upgrade.
* Check some of the following command outputs:
  * install verify packages
  * show logging
  * show install log detail
  * show alarms
