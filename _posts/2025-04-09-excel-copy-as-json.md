---
title: "Excel: Copy as JSON"
slug: "excel-copy-as-json"
date: "2025-04-09T12:01:46"
categories: ["Uncategorized"]
tags: ["code", "excel"]
---
A new twist on my existing [Copy as CSV]({{ site.baseurl }}{% link copy-as-csv.md %}) macro, this code will copy the selected cells in an Excel spreadsheet as a JSON document to your clipboard.

<!--more-->

![image]({{ "/assets/img/posts/2025-04-09-copy-as-json.gif" | relative_url }})

Install this in Excel using the same instructions as provided in my 
[Copy as CSV]({{ site.baseurl }}{% link copy-as-csv.md %}) post, but just use the below code instead. It will return a list of dicts, with keys based on the first row of the selected cells. Each row in the selection becomes an item in the list. All values are treated as strings for simplicity.

{% gist a8f9a31440c74cbf9112f2a1ccf7f04f %}