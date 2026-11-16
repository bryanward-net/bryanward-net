---
title: "Junos DCD commit check failed"
slug: "junos-dcd-commit-check-failed"
date: "2023-11-01T08:25:17"
categories: ["Uncategorized"]
tags: ["juniper", "junos"]
---
There are likely going to be other reasons why you may get an error upon committing a config such as:

<!--more-->

`CommitError(edit_path: DCD commit check failed, bad_element: None, message: error: DCD commit check process dumped core error: configuration check-out failed)`  
or  
`dcd[16947]: ../../../../../../src/junos/sbin/dcd/infra/db-param/param_access.c:dcd_param_set_iff_ptr_value:2248: insist ‘len <= iff_obj_maxlen[param_ix]’ failed`  
but here’s one that I encountered and found a fix for.

This post is mainly search-engine food, as there were no useful results when I began troubleshooting.

The issue relates to [PR1696428](https://prsearch.juniper.net/problemreport/PR1696428) where an interface has more than 256 VLAN members tagged, and they are tagged using VLAN Names instead of VLAN IDs. If you tag the VLANs using IDs, this problem does not occur.
