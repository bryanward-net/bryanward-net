---
title: "How Much Downtime?"
slug: "how-much-downtime"
date: "2020-02-27T12:02:21"
categories: ["Uncategorized"]
tags: ["mist"]
---
We’ve recently begun to post our maintenance windows as “Network Updates” instead of “Network Outages”. At a Layer 8 this makes sense, keeps the bosses and the users all warm and fuzzy.

<!--more-->

But I care about Layer 1. I always knew Mist APs were lightning quick to do firmware updates, but today I decided to quantify that.

Here we have a Mist AP43 upgrading between minor builds within the 0.6 firmware train. How long should I expect/plan for the WiFi to be down, err updating, for?

15 seconds.

Yup, it takes 15 seconds of potential\* user interruption to update Mist AP43. Here’s some proof:

![image]({{ "/assets/img/posts/2020-02-27-image1.png" | relative_url }})

This is a Wireshark IO Graph showing just Beacon frames from the AP I was upgrading.

![image]({{ "/assets/img/posts/2020-02-27-image2.png" | relative_url }})

And because PCAP or it didn’t happen…

Note the Time Delta of 14.9 seconds. That’s the time from the last Beacon that was transmitted pre-update until the first beacon post-update. You’ll also notice that the Sequence Number (SN) counter has rolled over to 1 showing that yes, this is because the radio was reset, not because I missed frames in my capture.

Sure, the actual update process does take longer – the LED blinks various colors for a couple of minutes and it takes a couple more minutes for the GUI/API to reflect that the update is actually complete, but in terms of user-facing scheduled interruption (scheduled improvement?), it’s hardly noticeable.
