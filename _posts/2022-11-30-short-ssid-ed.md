---
title: "Short SSID-ed"
slug: "short-ssid-ed"
date: "2022-11-30T00:06:39"
categories: ["Uncategorized"]
tags: ["python", "rnr", "ssid", "wifi"]
---
While listening to [Clear to Send Episode 308 “Reduced Neighbor Report”](https://www.cleartosend.net/cts-308-reduced-neighbor-report/), Francois and Jiri posed a challenge to develop a Short SSID translator… Challenge Accepted!

<!--more-->

I strongly encourage you to listen to the episode.

What is a Short SSID? Well, it’s a new piece of an 802.11 Beacon that is designed to assist with roaming to the 6GHz (Wi-Fi 6E) band. SSIDs broadcasting on the usual 2.4GHz and 5GHz bands can include this information within the Reduced Neighbor Report information element to inform a client that a network exists in a band that it may not have yet scanned. Since there are [so many 6GHz channels](https://www.linkedin.com/pulse/updated-unlicensed-spectrum-charts-keith-parsons?trk=public_profile_article_view) available, this helps to reduce the time needed for off-channel scanning and provide a client device with known [good roaming candidates](https://www.extremenetworks.com/extreme-networks-blog/the-off-road-trail-to-6-ghz-ap-discovery/).

![image]({{ "/assets/img/posts/2022-11-30-image1.png" | relative_url }})

It’ll be just another second…

Ok, makes sense. It’s sort of like an implementation of [802.11k’s Neighbor Reports](https://mrncciew.com/2014/09/11/cwsp-802-11k-ap-assisted-roaming/), but included in the beacon. Why not just use the same mechanism as .11k does? Well, 802.11k only provides Neighbor Reports for neighboring BSSIDs and not for neighboring SSIDs. That’s fine if a client device is looking to roam within the same SSID, but it’s less useful for discovering other SSIDs than the one the client is currently associated to. (Why would you want to roam to another SSID, isn’t that generally undesired? Well, using another SSID for your 6GHz band may be [suggested](https://securityuncorked.com/2022/08/wifi-security-wpa2-vs-wpa3/) as a best practice as you transition to Wi-Fi 6E, which requires WPA3.) 802.11k Neighbor Reports make no mention at all of the SSID, only the neighboring BSSIDs and their channels. RNRs need to offer a mechanism by which a client can locate other SSIDs in addition to the Same SSID.

Alright, so why not provide the full SSID? Why does the RNR use a Short SSID? Well, I wish it provided the full SSID, but that’s not what the IEEE spec decided to do. The Short SSID Subfield is defined in 802.11-2020 Figure 9-632 as having a length of either 0 (not present) or 4 bytes. No more, no less. Sure, they could have made it 32 bytes (the maximum length for any SSID), put the full SSID in there, and then null-padded whatever may remain, but this spec was written with “High Efficiency” in mind after all…

> /\* Always 4 bytes for Short SSID \*/
>
> *The* [Wireshark source code](https://gitlab.com/wireshark/wireshark/-/raw/master/epan/dissectors/packet-ieee80211.c) *calls out the subfield size more clearly than the official spec*.

4 bytes == 32 bits. The Short SSID needs to be a compressed form of the full SSID… but how? Well, the 802.11-2020 spec tells you how!

IEEE 802.11-2020 9.4.2.170.3 defines how the Short SSID is calculated. If you want to see it for yourself, grab a free copy of the spec via the [IEEE GET](https://ieeexplore.ieee.org/browse/standards/get-program/page/series?id=68) site. I’ll save you the trouble, however. What this section describes, in an impressive combination of brevity, vagueness, and esoteric mathematical complexity, is that you need to calculate the CRC-32 of the full SSID. CRC-32, by definition, yields a 32-bit long “fingerprint” of whatever input data is fed into the algorithm. (Fun fact, CRC-32 is also what’s used to calculate the FCS at the end of an Ethernet frame.) This makes it a perfect fit for those 4 bytes in the Short SSID subfield.

If you want to calculate CRC-32 values of your own, there are numerous ways to do that. The simplest is to use <https://crc32.online/>or another similar site. You can also use Python libraries like [binascii](https://docs.python.org/3/library/binascii.html#binascii.crc32) and [zlib](https://docs.python.org/3/library/zlib.html#zlib.crc32) to do this programmatically. Feeding in an SSID as the input data should yield the same 4-byte-long hexadecimal value as the RNR uses.

![image]({{ "/assets/img/posts/2022-11-30-image2.png" | relative_url }})

I got [the same value that Jiri Brejcha shared in his PCAPs](https://twitter.com/jiribrejcha/status/1591188679086571520?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1591188679086571520%7Ctwgr%5Ece44dc049df69be118c8f55cafa23f4780973ea4%7Ctwcon%5Es1_&ref_url=https%3A%2F%2Fwww.cleartosend.net%2Fcts-308-reduced-neighbor-report%2F) during his recording of the Clear to Send podcast!

Alright! So, now we know how to generate the Short SSID value. This alone isn’t of much use, but some software developers of Wi-Fi scanners and protocol analyzers may be able to use this to correlate neighbor BSSes.

What would be more useful would be to look at the Short SSID and determine the full SSID. Reversing a CRC-32 value is possible if the original source value is [4 bytes or less](https://stackoverflow.com/a/1514047). It’s also technically possible to reverse a CRC-32 value of a longer source value into a set of possible values that would produce the specified CRC value. The CRC-32 algorithm is not collision-free, so there is no way to guarantee what original value was used to generate any given CRC value.

That said, let’s do it anyway.

I’m now going to share with you [**a script I’ve made**](https://gist.github.com/bryanward-net/3787585868d2c14cadea9958e6145b4c), based on work by [theonlypwner](https://github.com/theonlypwner/crc32) and [Daniel Vik](http://www.danielvik.com/2013/07/rewinding-crc-calculating-crc-backwards.html). It has the ability to reverse a Short SSID into a list of possible full SSIDs. The results are long, it takes more than a few seconds, but it does give you the original value – you just have to find it in the list…

Example. I reversed 0x4FF8348A, the CRC-32 value for the SSID “Myopia”. It took 73.2 seconds for the script to complete. Over 75 thousand results were generated, but in that list of results, the original value was found within the first 10 results…

Another example. Using the same PCAP that Jiri referenced in the podcast, I reversed 0x492011DC into “Cisco 6”. The script took 72.5 seconds to run this time, and also produced over 75 thousand results. The original value was found closer to the 300th possible value.

I suspect that if someone were so inclined, they could (a) improve this script to handle up to 32 character long SSIDs, (b) support a wider character set (right now, I’ve only accounted for basic printable ASCII characters), and (c) apply some statistical analysis to filter out the returned possible values that are “unlikely” to be correct – maybe search for dictionary words.

What say you? Challenge Completed?
