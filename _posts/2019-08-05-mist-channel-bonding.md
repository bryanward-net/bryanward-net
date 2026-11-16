---
title: "Mist Channel Bonding"
slug: "mist-channel-bonding"
date: "2019-08-05T13:08:05"
categories: ["Uncategorized"]
tags: ["mist", "wifi"]
---
When using bonded channels in 5GHz (40MHz or 80MHz bandwidth), Mist RRM uses the lower channel of the bond as the primary. (ex, if bonding channels 44 and 48, it will select 44 as the primary channel).

However, if setting channels via API, you’ll want to make sure to only “allow” the lower channels, otherwise the AP might decide to use the upper channel as the primary.

Example, don’t set this in the radio\_config API:

```
"bandwidth": 40, "channel": 0, "channels": [36, 40, 44, 48, 149, 153, 157, 161]
```

But set this instead:

```
"bandwidth": 40, "channel": 0, "channels": [36, 44, 149, 157]
```

As this will ensure that only the lower channels are selected as the primary.

Setting “channel” to 0 allows the RRM algorithm to select the best channel from the list of “channels” specified.

Additionally, the API does not indicate both channels in a bond, but only the primary channel:

```
'band_5': {'bandwidth': 40, 'channel': 48, ...
```

This shows that the AP is using channels 44 and 48, but 48 is the primary.

After removing 48 from the allowed channels list, the API now shows:

```
 'band_5': {'bandwidth': 40, 'channel': 44, ...
```

The Mist GUI does show both channels in a bond, with the primary channel being in bold text.
