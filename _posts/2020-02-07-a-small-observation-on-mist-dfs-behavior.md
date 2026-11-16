---
title: "A Small Observation on Mist DFS Behavior"
slug: "a-small-observation-on-mist-dfs-behavior"
date: "2020-02-07T10:11:04"
categories: ["Uncategorized"]
tags: ["mist"]
---
Mist APs seem to maintain their channel width upon detecting a DFS event. They do not narrow down to 20MHz. They also do not all jump to channel 36 upon such an event, they seem to pick a next-best channel.

![](/assets/images/2020-02-07-image1.png)
![](/assets/images/2020-02-07-image2.png)
![](/assets/images/2020-02-07-image3.png)
