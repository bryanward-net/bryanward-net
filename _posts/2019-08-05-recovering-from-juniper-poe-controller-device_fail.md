---
title: "Recovering from Juniper PoE Controller DEVICE_FAIL"
slug: "recovering-from-juniper-poe-controller-device_fail"
date: "2019-08-05T13:04:37"
categories: ["Uncategorized"]
tags: ["juniper", "junos"]
---
As I install/upgrade a ton of new Juniper EX3400-48P edge switches in VCs, I’ve run into a few issues where upgrading Junos and then the PoE controller firmware results in the PoE Controller failing.

Following a Junos software upgrade, I issued the following command to upgrade the PoE controllers of the switches. (Why this is a separate step is beyond me.)

```
request system firmware upgrade poe fpc-slot all-members
```

The output of

```
show poe controller
```

would indicate that the upgrade is in progress, a status of “SW\_DOWNLOAD”. But then after upgrade is complete it would report a status of DEVICE\_FAIL on one or more members of a VC. Worse yet, after a reboot, the output of show poe controller would just be blank!

I had grown accustom to requesting hardware RMAs from Juniper, as that was what I was told to do on the first case I opened regarding this issue.

A few RMA-ed switches later, and the JTAC rep who picked up my most recent request for an RMA clued me into a process which would allow me to fix the issue myself, without an RMA. It’s a remote fix too!

Here’s the process (for Junos 18.1R3 at least):

> Power Cycle the affected FPC. Do not soft reboot.
>
> Once FPC joins the VC, Execute the below command from operational mode:
>
> `request system firmware upgrade poe fpc-slot 1 file /usr/libdata/poe_latest.s19`  
> Change the fpc-slot number accordingly.
>
> It’s recommended to push the PoE code one by one instead of all-members (if multiple FPC’s are affected)
>
> Check the status of the upgrade using “show poe controller”.
>
> Once the above command is executed and the upgrade completes, the FPC should automatically reboot. If not reboot from CLI.
>
> Once the FPC is online check the PoE version with this command “show chassis firmware detail”. PoE version should be latest 2.1.1.19.3
>
> If the version is correct, the PoE devices should work.

After issuing the upgrade command, it took a bit before “show poe controller” outputted anything. It would just return blank output, but then eventually did report SW\_DOWNLOAD. From there, everything was as expected with a normal PoE controller upgrade. I did end up rebooting the switch using the CLI here after the controller reported a status of “AT\_MODE”, which indicated a successful upgrade. Testing the switch with a Fluke Link Runner did show proper PoE power was being sent on the ports.

The note from JTAC about “It’s recommended to push the PoE code one by one instead of all-members” leads me to believe there’s a known issue when performing this upgrade on a multi-member VC. (Again, talking about Junos 18.1R3 here) In all instances where the poe controller upgrade has failed, it has been when upgrading multiple members in a VC. I’ve yet to see a switch fail it’s poe controller upgrade when being upgraded standalone or as a single-member VC.

Now, why would you want to upgrade the PoE controller in the first place? Well, any time you upgrade Junos, it’s recommended to perform this step. Again, why this isn’t just part of the normal Junos upgrade process is a bit bizarre, but hey, it is what it is. Also, the PoE controller upgrade is required for some switches to properly support 802.3at and enhanced 802.3af PoE. If you’ve got devices that need more power (WiFi APs anyone?) then you’ll want to make sure you do this step.

See here for official Juniper documentation:

<https://www.juniper.net/documentation/en_US/junos/topics/task/installation/poe-controller-software-upgrading.html>
