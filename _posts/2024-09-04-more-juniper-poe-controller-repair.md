---
title: "More Juniper PoE Controller Repair"
slug: "more-juniper-poe-controller-repair"
date: "2024-09-04T10:21:35"
categories: ["Uncategorized"]
tags: ["juniper", "junos", "poe"]
---
This is a continuance of my earlier article “[Recovering from Juniper PoE Controller DEVICE\_FAIL](https://bryanward.net/wp/2019/08/05/recovering-from-juniper-poe-controller-device_fail/)“.

<!--more-->

Due to chip supply constraints, Juniper has had to swap the PoE controller chip in some of their switches.  
The different chip requires different firmware, and so the file `/usr/lib/poe_latest.s19` may no longer exist.  
Newer versions of Junos are required to use this different model PoE chip as well.

```
bward@4300mp> request system firmware upgrade poe fpc-slot 1 file /usr/libdata/poe_latest.s19  
PoE: cannot open file
```

See related [TSB71559](https://kb.juniper.net/TSB71559), and PRs [PR1744343](https://prsearch.juniper.net/problemreport/PR1744343), [PR1741298](https://prsearch.juniper.net/problemreport/PR1741298), [PR1745088](https://prsearch.juniper.net/problemreport/PR1745088), and [PR1743547](https://prsearch.juniper.net/problemreport/PR1743547)

If you need to manually apply firmware to the PoE controller on a switch (such as when the output of show poe controller reports DEVICE\_FAIL, or if the PoE Controller simply isn’t listed), you’ll now need to load the model-specific firmware file:

```
root@4300mp:RE:0% cd /usr/libdata  
root@4300mp:RE:0% ls -la  
…  
lrwxr-xr-x 1 root wheel 32 Sep 4 03:26 poe_at_type1.s19 -> /var/host/poe_msemi_69200_at.s19  
lrwxr-xr-x 1 root wheel 32 Sep 4 03:26 poe_at_type2.s19 -> /var/host/poe_msemi_69220_at.s19  
lrwxr-xr-x 1 root wheel 32 Sep 4 03:26 poe_bt_type1.s19 -> /var/host/poe_msemi_69200_bt.s19  
lrwxr-xr-x 1 root wheel 32 Sep 4 03:26 poe_bt_type2.s19 -> /var/host/poe_msemi_69220_bt.s19  
…
```

The Type1 files are for the older model PoE controller chip, whereas the Type2 files are for the newer model.  
You’ll also need to choose between AT and BT mode, as appropriate.

The command you’ll need to use looks something like:  
`request system firmware upgrade poe fpc-slot 1 file /usr/libdata/poe_bt_type1.s19`

**Don’t forget to reboot after doing this!**

But, how do you know if your switch has a Type1 or Type2 PoE controller?  
Well, if the PoE controller is working… you can see this in the output of `show chassis firmware detail`

```
bward@4300mp> show chassis firmware detail
FPC 0
PoE firmware 3.5.2.0.30 (type1)
PHY version 5d47.37a.47.305
Boot Firmware
U-Boot ***
Boot Firmware
loader FreeBSD/i386 bootstrap loader 1.2
```

But if it’s not working, then the PoE firmware line won’t be present 🙁

I cannot confirm the below, as I no longer have a failed device, but it looks like you can determine the type of PoE controller by using the following:

```
bward@4300mp> start shell user root

root@4300mp:RE:0% vty fpc1

FPC1(4300mp vty)# sh poe bt version
REQUEST : 0x02 0x17 0x07 0x1E 0x21 0x4E 0x4E 0x4E 0x4E 0x4E 0x4E 0x4E 0x4E
RESPONSE: 0X03 0X17 0X00 0X4E 0X18 0X01 0X60 0X00 0X1E 0X03 0XE8 0X4E 0X4E
HW version = 0x00
Product Number = 0x18
Controller Type = Type1(PD69200)-BT
Full SW version = 24.0352.30.00
Internal SW = 0x03e8

FPC1(4300mp vty)# exit

root@4300mp:RE:0% exit
```

If you don’t have an 802.3bt capable device, then remove the `bt` keyword from the above command.

UPDATE

It appears that a switch will produce an error if you try to load the wrong firmware:

```
bward@4300mp> request system firmware upgrade poe fpc-slot 1 file /usr/libdata/poe_bt_type1.s19
PoE: file format/size/content corrupted

{master:0}
bward@4300mp> request system firmware upgrade poe fpc-slot 1 file /usr/libdata/poe_bt_type2.s19   
Firmware upgrade initiated. Poe Upgrade takes about 10 minutes
Use 'show poe controller' to get the download status,
Please Reboot the system after Upgrade is complete.
```
