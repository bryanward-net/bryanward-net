---
title: "dartmist  – A Python Library for the Mist API"
slug: "dartmist-a-python-library-for-the-mist-api"
date: "2020-04-10T08:39:03"
categories: ["Uncategorized"]
tags: ["code", "mist"]
---
Today, I am pleased to announce the first public release of [dartmist](https://www.bryanward.net/dartmist) [– A Python Library for the Mist API.](https://www.bryanward.net/dartmist)

This library is, hopefully, a friendly way to interact with the Mist API. This was written by network engineers for network engineers. Features will be added to it over time.

Additionally, I am releasing my [rolling-update](https://github.com/bryanward-net/mist-rolling-update) script for updating firmware on Mist APs, one at a time. I’ve been using variations of this script in production for a while now, and as I type this, it is churning away at a couple more sites in production. It is a definite time-saver!

The goal of rolling-update is simple: pick an AP in a given site, update it’s FW, wait for it to come back online, and then move onto another AP. By going through APs one-by-one like this, coverage holes are completely minimized. Only a single AP is ever offline for an update at any given time. [Users will hardly notice the outage](https://www.bryanward.net/wp/2020/02/27/how-much-downtime/). Because of this, I’ve been able to do firmware updates during business hours! Also, since it’s a simple python script, it can be scheduled as a cron or at job and run overnight or on a weekend. In order to accomplish this via the Mist GUI, you’d have to manually update each AP in a site by selecting the AP, clicking Update, picking the FW version, and then waiting for the update to complete by mashing the refresh button every 60 seconds. Sure, you can update multiple APs at a time via the GUI, but then you’re creating larger coverage holes.

The other key thing is that the script waits for the AP to return to the Connected status and reporting the new FW version. This means that the script will stall if an AP fails to update and come back online. This is intentional! In the event of a failure, you’re only ever left with one single AP down instead of a whole site or campus worth of APs as you’d have with an update of controller-based system! (#NeverLookingBack)

If either dartmist or rolling-update are of use to you and you end up giving them a shot, let me know – I’d be honored for others to use my work in their environments. I’m open to any/all feedback.
