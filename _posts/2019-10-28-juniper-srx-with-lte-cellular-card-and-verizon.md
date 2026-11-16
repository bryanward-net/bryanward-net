---
title: "Juniper SRX with LTE Cellular Card and Verizon"
slug: "juniper-srx-with-lte-cellular-card-and-verizon"
date: "2019-10-28T16:27:01"
categories: ["Uncategorized"]
tags: ["cellular", "juniper", "junos", "LTE", "sim card", "verizon"]
---
The [Juniper SRX320](https://www.juniper.net/us/en/products-services/security/srx-series/datasheets/1000550.page) is a pretty powerful little firewall (and router, switch, etc.). I mean, just listen to that fan when it boots up! Or take a look at the AC adapter! That’s some power! (Well-known fruit brand charger shown for scale.)

<!--more-->

![image]({{ "/assets/img/posts/2019-10-28-image1.png" | relative_url }})

But it’s actually an awesome device. It has:

2x SFP/SFP+ cages, 6x 1G Copper interfaces with PoE+, and can take up to 2 Mini-PIM WAN modules.

Here, I’m using it with the [SRX-MP-LTE-AE module](https://www.juniper.net/documentation/en_US/release-independent/junos/topics/topic-map/mpim-lte.html) to connect to Verizon LTE in the US for it’s primary WAN uplink. I plan to establish an tunnel of some sort back to the main campus network via this link.

Configuring the device to use the LTE modem as it’s uplink was very easy. Juniper’s docs for [Configuring the LTE Mini-PIM](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/security-interface-config-lte-interfaces.html#id-configuring-the-lte-mini-pim-as-the-primary-interface) are pretty straightforward, even if not 100% applicable to my project. I’ll boil it down to the bare minimum needed for LTE as the primary data service here.

```
configure

delete interfaces interface-name unit 0 backup-options interface dl0.0
set interfaces dl0.0 family inet negotiate-address
set interfaces dl0 unit 0 dialer-options always-on

run request modem wireless create-profile profile-id 1 cl-1/0/0 slot 1 access-point-name VZWINTERNET authentication-method none
set interfaces cl-1/0/0 act-sim 1
set interfaces cl-1/0/0 cellular-options sim 1 select-profile profile-id 1
set interfaces cl-1/0/0 cellular-options sim 1 radio-access automatic

commit
exit

show modem wireless profiles cl-1/0/0 slot 1
show modem wireless network
show interfaces dl0.0
```

cl-1/0/0 is the physical modem interface and dl0 is the dialer interface. This seems to be a holdover from a long time ago when multiple modems were grouped together into dialer pools to allow simultaneous connections to take place.

In short, dl0.0 is where your WAN IP Address will end up. Assuming the config was entered properly, and your SIM card is activated by the provider, you should be able to route IPv4 to the internet and resolve DNS names within a minute or two after committing the config.

If you can ping the default gateway, but nothing beyond that, then chances are your SIM card is not activated/provisioned by the provider yet. Being able to ping the default gateway is a great test of the modem configuration before actually spending money on a data plan.

And on that note, let’s talk about how to get Verizon to actually provision a SIM card for this device…

Let’s start with the fact that not all SIM cards are the same, even if from the same provider. For example, I needed a brand new 4G LTE SIM card from Verizon when I upgraded from my Samsung Galaxy S5 to my OnePlus 6T. But, I did not need a new SIM card when upgrading from the Galaxy Nexus to the S5. Sometimes, they’re the same, sometimes they’re not. iPhones get yet an even different SIM card, not to even begin to mention the issues with iMessage interrupting SMS/MMS messages destined for a phone number that was previously on an iPhone…

The LTE Mini-PIM card needs SIM model DFILLSIM-TRI-A – at least that’s what Verizon calls it. Verizon does [list a few other supported/required SIM cards](https://opendevelopment.verizonwireless.com/device-showcase/device/5589), but the DFILLSIM-TRI-A is the one I ordered for this and is the one that is currently working for me. The Mini-PIM card takes a full-size SIM card, not a mini/nano/micro size. It does come with the size adapters in the box, so don’t worry if you don’t have a full-size one sent to you for some reason.

When ordering a SIM for this over the phone with Verizon customer support, they’ll want to check the IMEI number of your device to confirm SIM compatibility. The IMEI number is printed on the top of the radio module as seen in the picture below, but it can also be accessed via command line with the following command:

```
> show modem wireless network cl-1/0/0
LTE Connection details
...
Wireless Modem Network Info
...
International Mobile Equipment Identification (IMEI/MEID): XXXXXXXXXXXXXXX
...
```

![image]({{ "/assets/img/posts/2019-10-28-image2.png" | relative_url }})

Verizon will not activate a data plan or assign a line to an account until you have both the SIM card and device in hand. So, wait a couple of days for the SIM card to arrive in the mail. From Verizon, SIM cards and standard shipping are always free. Don’t let them tell you otherwise. You can also pick up a SIM card from a Verizon corporate store.

Once you have the SIM card and your Juniper kit ready, give Verizon customer support a call and ask them to add a new line to your account.

They’ll want the SIM Card number (ICCID) and the Device IMEI number. At this point, you should let them know that you’ll be wanting them to do a SIM-Only activation, not a SIM-and-Device activation. If they seem confused, have them transfer you to advanced tech support now. The Mini-PIM cards are obviously not your typical smartphone or Hotspot device, so Tier 1 support likely lacks the access to provision these devices. Tier 2 support also likely has no idea what it is you’re trying to do, but can at least do what is needed on their end to make it work.

As far as plans go, make sure they sell you a Data Only plan. This device will likely not be making any phone calls or sending/receiving SMS messages, so don’t pay for that. Yes, Data Only plans do exist. I chose a 2GB per month data plan with no deprioritization (throttling). Usage beyond 2GB per month would be billed at a per-gigabyte rate, but the service would not be shut off. (I won’t discuss specific prices here, but they were fair – comparable to the “Small” 2GB plan currently advertised at $35/month before any business discounts (28 Oct 2019) <https://www.verizonwireless.com/plans/> .) The plan I chose is called “Nationwide for Business Data Share 2GB”. Ask the person on the phone for plan code 87211 to cut to the chase. Or have them look that one up first and then adjust your usage needs accordingly.

Once they do what they need to, they should assign your device a phone number. This won’t be used by your device, but it’s important to note for billing/accounting purposes.

If they run into issues provisioning the SIM Card, ask that they make sure that the SIM was “released” in their system before trying to provision it again. This, for me, resolved the error on their system of “SIM IS NOT IN AA STATUS” or “SIM is not in Activated Status”.

You might also want to power off the SRX or shut off the radios on the modem card while they do their backend provisioning. It shouldn’t matter nowadays, but my history with Verizon provisioning shows that having the device powered off used to be a critical step so it might still be important today.

To shut down the modem radios (akin to Airplane Mode), issue the following command:

```
configure
set interfaces cl-1/0/0 disable
commit
```

To turn them back on, do the following:

```
configure
delete interfaces cl-1/0/0 disable
commit
```

Once they tell you that you’re all set, have them stay on the line for a minute while you confirm the modem comes up and you can route to the Internet.

If everything is good, then you should see the following sort of information:

```
> show modem wireless network cl-1/0/0
LTE Connection details
  Connected time: xxx
  IP: x.x.x.x
  Gateway: x.x.x.x
  DNS: x.x.x.x
...

> show interfaces dl0.0
  Logical interface dl0.0 (Index 85) (SNMP ifIndex 502)
    Flags: Up Point-To-Point SNMP-Traps 0x0 Encapsulation: ENET2
    Dialer:
      State: Active, Dial pool: 1
      Dial strings: 1234
      Subordinate interfaces: cl-1/0/0 (Index 151)
      Activation delay: 0, Deactivation delay: 0
      Initial route check delay: 120
      Redial delay: 3
      Callback wait period: 5
      Load threshold: 0, Load interval: 60
    Bandwidth: 300mbps
    Input packets : 39
    Output packets: 58
    Security: Zone: untrust
    Allowed host-inbound traffic : tftp
    Protocol inet, MTU: 1414
      Flags: Sendbcast-pkt-to-re, Negotiate-Address
      Addresses, Flags: Kernel Is-Preferred Is-Primary
        Destination: x.x.x.x/xx, Local: x.x.x.x,
        Broadcast: x.x.x.x
    Protocol inet6, MTU: 1414
    Max nh cache: 100000, New hold nh limit: 100000, Curr nh cnt: 1,
    Curr new hold cnt: 1, NH drop cnt: 0
      Flags: Negotiate-Address
      Addresses, Flags: Is-Default Is-Preferred Is-Primary
        Destination: x:x:x:x::/xx,
        Local: x:x:x:x:x:x:x:x
      Addresses, Flags: Is-Preferred
        Destination: fe80::/64, Local: fe80::x:x:x:x
```

Another thing to note is the Modem Firmware. Chances are, the Mini-PIM card comes with some ancient firmware loaded. Using the latest firmware available will allow the radios to support more radio bands and roam better throughout the cell provider’s RAN. To upgrade the modem using the Free-Over-the-Air upgrade offered by the provider (recommended, as this downloads the latest firmware supported by your provider), run the following commands:

```
show modem wireless firmware cl-1/0/0

LTE mPIM firmware details
  Product name: Junos LTE mPIM
  Serial number: XXXXXXXXXX
  Hardware version: AcceleratedConcepts/Juniper 06
  Firmware version: 17.5.108
  MAC: xx:xx:xx:xx:xx:xx
  System uptime: 1134 seconds
Wireless modem firmware details
  Modem firmware version: 9999999_9904780_SWI9X30C_02.20.03.22_00_VERIZON_002.026_001
  Modem Firmware build date: 11/10/2016
  Card type: MC7455
  Modem manufacturer: Sierra Wireless, Inc
  Hardware version: 1.0
  Power & Temperature: Normal 3337 mV, Normal 32.00 C
OTA status
  State: Disabled
  New firmware available: No
Number of SIM: 1
Slot of active: 1
Status of SIM 1
  SIM state: SIM present
  Modem PIN security status: Disabled
  SIM status: SIM Okay
  SIM user operation needed: No Op
  Retries remaining: 3
```

*Note that Modem Firmware build date from 2016!*

```
> request modem wireless fota enable cl-1/0/0
Set FOTA on modem succeeded

> request modem wireless upgrade cl-1/0/0
Launch FOTA upgrade succeeded
```

Then wait a minute and run:

```
> show modem wireless firmware cl-1/0/0
LTE mPIM firmware details
  Product name: Junos LTE mPIM
  Serial number: XXXXXXXXXX
  Hardware version: AcceleratedConcepts/Juniper 06
  Firmware version: 17.5.108
  MAC: xx:xx:xx:xx:xx:xx
  System uptime: 1262 seconds
Wireless modem firmware details
  Modem firmware version: 9999999_9904780_SWI9X30C_02.20.03.22_00_VERIZON_002.026_001
  Modem Firmware build date: 11/10/2016
  Card type: MC7455
  Modem manufacturer: Sierra Wireless, Inc
  Hardware version: 1.0
  Power & Temperature: N/A
OTA status
  State: Enabled
  New firmware available: Yes
  New firmware version: 02.30.01.01_Verizon_002.052_001
Number of SIM: 0
Slot of active: 1
```

Note the new OTA Status showing new FW available. After the FW downloads (about a minute or two) all the LEDs on the front of the Mini-PIM card will blink while the FW is flashed. This takes about 5 minutes. Then, the card will come back online with the new FW installed. (The latest is apparently from 2018…) You might also need/want to reboot the whole system at this point too.

```
> show modem wireless firmware cl-1/0/0
LTE mPIM firmware details
  Product name: Junos LTE mPIM
  Serial number: XXXXXXXXXX
  Hardware version: AcceleratedConcepts/Juniper 06
  Firmware version: 17.5.108
  MAC: xx:xx:xx:xx:xx:xx
  System uptime: 1810 seconds
Wireless modem firmware details
  Modem firmware version: 9999999_9904780_SWI9X30C_02.30.01.01_00_VERIZON_002.052_001
  Modem Firmware build date: 13/07/2018
  Card type: MC7455
  Modem manufacturer: Sierra Wireless, Inc
  Hardware version: 1.0
  Power & Temperature: Normal 3341 mV, Normal 33.00 C
OTA status
  State: Enabled
  New firmware available: No
Number of SIM: 1
Slot of active: 1
Status of SIM 1
  SIM state: SIM present
  Modem PIN security status: Disabled
  SIM status: SIM Okay
  SIM user operation needed: No Op
  Retries remaining: 3
```

Needless to say, **this is a service-interrupting upgrade**. So, it’s best to disable future automatic FOTA updates:

```
> request modem wireless fota disable cl-1/0/0
Set FOTA on modem succeeded
```
