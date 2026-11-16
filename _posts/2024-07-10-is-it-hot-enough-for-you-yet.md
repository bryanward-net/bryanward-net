---
title: "Is it hot enough for you yet?"
slug: "is-it-hot-enough-for-you-yet"
date: "2024-07-10T12:22:04"
categories: ["Uncategorized"]
tags: ["juniper", "junos", "monitoring", "temperature"]
---
Today’s post concerns temperature thresholds for telecom facilities, and the rabbit hole I discovered when figuring out how to monitor temperature of Juniper equipment.

<!--more-->

If you read the datasheets for the Juniper [EX series](https://www.juniper.net/documentation/us/en/hardware/ex4300/topics/topic-map/ex4300-site-guidelines-requirements.html#ex-series-environment__d22e33) or QFX series ([10002](https://www.juniper.net/documentation/us/en/hardware/qfx10002/topics/topic-map/qfx10002-site-guidelines-requirements.html#qfx10002-environmental__d59e33), [5120](https://www.juniper.net/documentation/us/en/hardware/qfx5120/topics/topic-map/qfx5120-site-guidelines-requirements.html)), you’ll find that this equipment likes to operate in an environment “in temperature range of 32°F through 104°F (0°C through 40°C)”. Some models push that as high as 45°C. These are actually pretty generous ranges, especially considering that the American Society of Heating, Refrigerating and Air-Conditioning Engineers (ASHRAE) – an organization that publishes standards for the HVAC industry, much like the IEEE or BICSI does for us networking folks – says that [telecom facilities should have a design target between 18 and 27°C](https://xp20.ashrae.org/datacom1_4th/ReferenceCard.pdf). Knowing that that range is laughable for most Telecom Rooms (and even some datacenters), ASHRAE goes on to provide alternate thresholds based on the type of equipment. Most networking equipment would fall under Class A3, which only requires a temperature range of 5°C to 40°C, or Class A4, which is 5°C to 45°C. It’s almost like Juniper knew this when they designed their products and published their datasheets…

The motivation to write this article came about at 5PM yesterday, and again at 6AM this morning, when my phone blew up with alarms about some of my production equipment exceeding their temperature warnings.

The shame is entirely on me. I had been adding some rules to our production monitoring system yesterday, and configured the temperature thresholds based on the values published in Juniper’s datasheets (40 and 45°C, depending on the model). The problem is that the sensors inside the equipment do not measure ambient temperature – they’re located in the chassis in places that, even during normal operation in a properly cooled environment, will regularly exceed those limits.

So, how then can you determine which thresholds are actually appropriate based on the equipment and the sensors within? Let’s enter the rabbit hole… Well, unsurprisingly, there’s a command for that. I had expected the threshold values to appear in the output of `show chassis environment`, but they do not – only the present operating values appear here, as such:

```
> show chassis environment 
Class Item                           Status     Measurement
Power FPC 0 Power Supply 0           OK        
      FPC 0 Power Supply 1           OK        
Temp  FPC 0 PSU Sensor               OK         38 degrees C / 100 degrees F
      FPC 0 MGE-PHY Sensor           OK         42 degrees C / 107 degrees F
      FPC 0 DC-DC Sensor             OK         46 degrees C / 114 degrees F
      FPC 0 UM CONN Sensor           OK         36 degrees C / 96 degrees F
      FPC 0 CPU Sensor               OK         41 degrees C / 105 degrees F
      FPC 0 MAC Sensor               OK         45 degrees C / 113 degrees F
Fans  FPC 0 Fan Tray 0 Fan 0         OK         Spinning at normal speed
      FPC 0 Fan Tray 0 Fan 1         OK         Spinning at normal speed
      FPC 0 Fan Tray 1 Fan 0         OK         Spinning at normal speed
      FPC 0 Fan Tray 1 Fan 1         OK         Spinning at normal speed

> show chassis environment fpc 0 
FPC 0 status:
  State                      Online
  Temperature                             38 degrees C / 100 degrees F
```

Notice how dangerously close these values are to the datasheet limits of 40 and/or 45°C! No wonder my alarms all started to go off!

It turns out, there’s another command I should have looked at: `show chassis temperature-thresholds`

```
> show chassis temperature-thresholds 
                                        Fan speed      Yellow alarm      Red alarm      Fire Shutdown
                                       (degrees C)      (degrees C)     (degrees C)      (degrees C)
Item                                  Normal  High   Normal  Bad fan   Normal  Bad fan     Normal
FPC 0 PSU Sensor                          40    61       64       64       68       68         73
FPC 0 MGE-PHY Sensor                      48    68       71       71       75       75         80
FPC 0 DC-DC Sensor                        45    66       69       69       73       73         78
FPC 0 UM CONN Sensor                      44    64       67       67       71       71         76
FPC 0 CPU Sensor                          47    67       70       70       74       74         80
FPC 0 MAC Sensor                          50    71       73       73       76       76         80
```

That’s much more reasonable! The limits there are clearly displayed for each sensor, including several tiers of alarm.

There is an [old post from Steve Puluka](https://community.juniper.net/discussion/changing-temperature-thresholds-on-ex-switch) that clued me into the presence of a hidden command to change these built-in alarm thresholds, however it’s best to not use it. These thresholds have been factory-set based on the capabilities of the equipment. If you’re curious though, here is it:

```
# set chassis temperature-threshold ?
Possible completions:
+ apply-groups         Groups from which to inherit configuration data
+ apply-groups-except  Don't inherit configuration data from these groups
  fire-shutdown        Threshold at which router will be shutdown within 10 seconds (degrees C)
  red-alarm            Threshold at which red alarm is set (degrees C)
  red-alarm-if-failed-fan  Threshold at which red alarm is set when bad fan present (degrees C)
  yellow-alarm         Threshold at which yellow alarm is set (degrees C)
  yellow-alarm-if-failed-fan  Threshold at which yellow alarm is set when bad fan present (degrees C)
```

If you actually leverage the built-in “alarm” status of your equipment, and you’re operating in extreme environments, you may find that you want to adjust these. The thread that I linked to goes on to say that this command only affects logging and not SNMP traps, and there’s no guarantee that it will properly trigger the built-in “alarm” status either. So, YMMV, and again my recommendation is to not mess with this. It’s best to fix the cooling situation in your facility than to change these thresholds.

Alright, what if you’re not using the built-in alarm status, and you want to query each sensor directly in your NMS? Unsurprisingly, there are [SNMP OIDs to do just that](https://supportportal.juniper.net/s/article/EX-How-to-check-temperature-CPU-memory-usage-by-SNMP-OID?language=en_US). The easiest one to use is `jnxOperatingTemp`, or `1.3.6.1.4.1.2636.3.1.13.1.7` . This corresponds to the output of `show chassis environment fpc` .

One more note. You should be monitoring the FPCs and not the REs. If you monitor just the REs, then you will not get the status of any line-cards in a chassis, only the master and backup RE, and could miss events occurring on just the line-cards. To put this in SNMP terms:  
`1.3.6.1.4.1.2636.3.1.13.1.7.7.1.0.0` is `show chassis environment fpc 0`   
`1.3.6.1.4.1.2636.3.1.13.1.7.7.2.0.0` is `show chassis environment fpc 1`   
`1.3.6.1.4.1.2636.3.1.13.1.7.7.3.0.0` is `show chassis environment fpc 2`   
etc.

`1.3.6.1.4.1.2636.3.1.13.1.7.9.1.0.0` is `show chassis environment re 0` and `1.3.6.1.4.1.2636.3.1.13.1.7.9.2.0.0` is `show chassis environment re 1`.

If you’re using the [RFC 6241 NETCONF](https://datatracker.ietf.org/doc/html/rfc6241) API to do your monitoring (which is worth looking into), you can use the following RPC to get the same data:

```
<get-environment-fpc-information>
    <fpc-slot>0</fpc-slot>
</get-environment-fpc-information>
```

I had hoped to stop writing at this point, but the thresholds for this measurement point are NOT in the output of `show chassis temperature-thresholds` ! So, what temperatures should be cause for concern when looking at this sensor?

Well, it seems there’s a command for that too, but it doesn’t seem to work on any platform I’ve tried!

```
> show chassis temperature-thresholds fpc 0     
error: command is not valid on the ex3400-48p

> show chassis temperature-thresholds fpc 0 
error: command is not valid on the ex4300-48mp

> show chassis temperature-thresholds fpc 0 
error: command is not valid on the qfx10002-72q
```

I also tried walking the entire SNMP MIB tree to see if there’s any OID or value that makes sense for an upper limit to jnxOperatingTemp. Long story short, there aren’t any OIDs that specify the thresholds of jnxOperatingTemp.

Juniper’s [RMON documentation](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/monitoring-network-service-quality-by-using-rmon.html#id-measuring-health__d130e23) for this measurement only says “Allowable range: To be baselined”, which isn’t very useful either.

Juniper’s HealthBot actually does make an [attempt at defining a threshold](https://github.com/Juniper/healthbot-rules/blob/master/juniper_official/Chassis/README.md#rule-name-check-chassis-temperature) though. It will trigger a yellow alert if the temperature rises above 45°C, and a red alert if it rises above 55°C. HealthBot’s logic does not take the hardware model into account, and it’s not entirely clear which measurement it is looking at.

Some other engineers like [Lindsay](https://lkhill.com/mx-upgrade-overheat/) and [Jon](https://51sec.weebly.com/blog/using-snmpv3-monitoring-juniper-srx-240h-alarm-andtemperature) have published their own articles detailing their woes of troubleshooting Juniper temperature issues. They make reference to Junos logging alarms when the temperature reaches 50°C. Lindsay’s article also reveals the scary fact that Junos will shut down some types of SFP modules at this 50°C limit as well. This shutdown behavior is related to the CHASSISD\_FPC\_OPTICS\_HOT\_NOTICE log message. Other similar nmemonics include CHASSISD\_TEMP\_HOT\_NOTICE, CHASSISD\_FRU\_HIGH\_TEMP\_CONDITION, and the dreaded CHASSISD\_OVER\_TEMP\_SHUTDOWN\_TIME at which the entire router/switch will shut down.

So, if Junos shuts things down at 50°C, I think HealthBot’s threshold of 55°C is far too high. On the other end, as I experienced in my environment, a facility ambient temperature of 19°C (66°F) – quite chilly – still had EX switches reporting an FPC temperature of over 40°C. In another much warmer facility, I have a QFX10k2 router reporting an FPC operating temperature of 55°C, and no SFPs have been shut down yet. Since I’ve yet to find an authoritative answer on what the limit for jnxOperatingTemp should be, I would think having my NMS alert at somewhere between 45 and 49°C is a safe starting point.

But I wanted to take a more precise approach. I decided that I wouldn’t alert on jnxOperatingTemp, but would instead alert on the FPC CPU Temp, which does has defined alarm thresholds as discussed earlier here. One problem though – there is no SNMP OID for FPC CPU Temp! It is available via NETCONF at `fpc-information/fpc/temperature` though…

Aha! That generic use of “fpc/temperature” lead me to discover that what SNMP calls jnxOperatingTemp is actually the same as FPC CPU Temp for the EX3400 series, which has defined thresholds as seen in the output of `show chassis temperature-thresholds`!

So, there are, in fact, defined thresholds for jnxOperatingTemp – they’re just displayed as something else! Doh!

What are they displayed as then? It gets worse, as that depends on the model, and sadly here the logic could use some work. Rather than consistently using the CPU Temp regardless of platform, it just picks the first temperature sensor listed in the output of `show chassis environment` to use as the value for the generic FPC Temperature measurement. What that means is that the NETCONF value for fpc/temperature is the following:

|  |  |  |  |
| --- | --- | --- | --- |
| **Platform** | **fpc/temperature Measurement Source** | **Yellow Alarm** | **SNMP OID** |
| EX2300C | FPC CPU Sensor | 65 | 1.3.6.1.4.1.2636.3.1.13.1.7.x.0.0 |
| EX3400 | FPC CPU Temp | 63 | 1.3.6.1.4.1.2636.3.1.13.1.7.x.0.0 |
| EX4300MP | FPC PSU Sensor | 64 | 1.3.6.1.4.1.2636.3.1.13.1.7.x.0.0 |
| EX4400 | FPC Main Board Sensor 1 | 69 | 1.3.6.1.4.1.2636.3.1.13.1.**9**.x.0.0 (.**7**.x.0.0 appears to be the CPU Die Temp Sensor) |
| QFX5110 | FPC Sensor TopLeft I | 69 | 1.3.6.1.4.1.2636.3.1.13.1.**9**.x.0.0 (I cannot determine where .7.x.0.0 gets its value from.) |
| QFX10002 | FPC Intake Temp Sensor | 65 | 1.3.6.1.4.1.2636.3.1.13.1.**9**.x.0.0 (.**7**.x.0.0 appears to be the CPU Die Temp Sensor) |

You’ll see that these also don’t line up nicely with the SNMP OIDs either., SNMP OID `1.3.6.1.4.1.2636.3.1.13.1.7.7` on the EX3400 platform is the same as the reported FPC CPU Temp. The same SNMP OID on the QFX10k2 platform is reported as the FPC CPU Die Temp, but `1.3.6.1.4.1.2636.3.1.13.1.7.9` on the QFX10k2 is the same as FPC Intake Temp. That .9 OID refers to the Routing Engine, which on a QFX10k2 I guess makes sense, as you can’t connect QFX10k2s into a virtual chassis… But you can connect EX4400s and QFX5ks into virtual chassis…

To recap: the EX3400 FPC CPU Temp and the QFX10k2 FPC Intake Temp are the values as shown in the command `show chassis environment fpc`. This same value is also shown in the output of the command `show chassis fpc`. This same measurement is also available via NETCONF RPC `get-fpc-information` and returns the value in `fpc-information/fpc/temperature`. The command `show chassis temperature-thresholds` lists the alarm limits for these values.

Since the alarm limits do vary by platform (and possibly by HW revision or FW version), I suggest you set your NMS to alarm based on the output of `show chassis temperature-thresholds` for your specific equipment, and make sure that it’s monitoring the correct SNMP OID or NETCONF RPC. After combing thru all the equipment in my environment, 63°C (145°F) of `fpc-information/fpc/temperature` seems like a very reasonable value at which I should be alerted and taking action. Some equipment has higher alarm limits, but I wouldn’t want anything to be running hotter than that anyway.
