---
title: "Excel: Copy as CSV"
slug: "excel-copy-as-csv"
date: "2020-01-17T13:53:13"
categories: ["Uncategorized"]
tags: ["code", "excel"]
---
Ever need to export cells to a CSV file? Tired of copy/pasting the cells you need into a new, blank sheet and exporting it as a CSV? Then being told you might lose some compatibility features, then needing to save-as a proper XLSX file again so you don't lose sheets when saving again? Try this!

<!--more-->

![image]({{ "/assets/img/posts/2020-01-17-copytocsv.gif" | relative_url }})

I assume you have Windows 10 and a modern version of Excel.

Right click on the Ribbon (toolbar) and select "Customize the Ribbon..."

![image]({{ "/assets/img/posts/2020-01-17-image-1.png" | relative_url }})
  
In the right-hand list, find Developer and make sure it's checked. Click OK.

![image]({{ "/assets/img/posts/2020-01-17-image-2.png" | relative_url }})

In the now visible Developer tab in the Ribbon, open Visual Basic, the first icon you see.

![image]({{ "/assets/img/posts/2020-01-17-image-3.png" | relative_url }})

![image]({{ "/assets/img/posts/2020-01-17-image-4.png" | relative_url }})

You should now see the Microsoft Visual Basic for Applications window appear.  You should also see a VBAProject (PERSONAL.XLSB) file loaded in the left-hand side of the editor.

If you don't, switch back to Excel and create a new, blank Excel file.  Make no changes and save it as

`C:\Users\_{{YOUR_NAME_HERE}}_\AppData\Roaming\Microsoft\Excel\XLSTART\PERSONAL.XLSB`

Make sure the file type is "Excel Binary Workbook (\*.xlsb)" and the filename is ALL CAPS.

![image]({{ "/assets/img/posts/2020-01-17-image-5.png" | relative_url }})

Now your editor should look similar to the following.

![image]({{ "/assets/img/posts/2020-01-17-image-6.png" | relative_url }})

If your file does not have a Module1 file, right click on the VBAProject item and choose Insert->Module

![image]({{ "/assets/img/posts/2020-01-17-image-7.png" | relative_url }})

Double-Click Module1 to open it in the right-hand editor.

A new blank editor will look like the following:

![image]({{ "/assets/img/posts/2020-01-17-image-8.png" | relative_url }})

You might already have some code in the editor. If so, just paste the below code at the end of your existing file, or create a new Module to keep things separated.

```
Public Sub CopyAsCSV()
 'https://www.bryanward.net/wp/2020/01/17/excel-copy-as-csv/
 'https://answers.microsoft.com/en-us/mac/forum/macoffice2011-macexcel/copy-cells-and-paste-them-as-csv/b8a05dd8-4605-408a-a1da-9842c767745f?msgId=1aa0503e-d369-4d6b-909f-c8679ee44e3c&tab=AllReplies&page=1
 'https://www.excelforum.com/excel-programming-vba-macros/353942-how-do-i-reference-dataobject.html

Dim oDO As DataObject
Dim rArea As Range
Dim i As Long
Dim j As Long
Dim nCount As Long
Dim sTxt As String

Dim useQuotes As Boolean
useQuotes = True


If TypeOf Selection Is Range Then
    With Selection
        For Each rArea In .Areas
            With rArea
                If useQuotes Then
                    For i = 1 To .Rows.Count
                        For j = 1 To .Columns.Count
                            If j = 1 Then
                                sTxt = sTxt + Chr(34) + .Cells(i, j).Text + Chr(34)
                            Else
                                sTxt = sTxt + "," + Chr(34) + .Cells(i, j).Text + Chr(34)
                            End If
                            nCount = nCount + 1
                        Next j
                        sTxt = sTxt + vbCrLf
                    Next i
                Else
                    For i = 1 To .Rows.Count
                        For j = 1 To .Columns.Count
                            If j = 1 Then
                                sTxt = sTxt + .Cells(i, j).Text
                            Else
                                sTxt = sTxt + "," + .Cells(i, j).Text
                            End If
                            nCount = nCount + 1
                        Next j
                        sTxt = sTxt + vbCrLf
                    Next i
                End If
            End With
        Next rArea
    End With
    Set oDO = New DataObject
    oDO.SetText sTxt
    oDO.PutInClipboard
End If
End Sub
```

While you're still here in the VBA editor, let's make sure the MSForms 2.0 library is loaded. It wasn't by default on my system and is required for this to work. Goto Tools>References... in the VBA Editor.

![image]({{ "/assets/img/posts/2020-01-17-image-9.png" | relative_url }})

Click Browse... and then in the Add Reference dialog, open the file `C:\Windows\System32\FM20.DLL` . It should already be installed on your system.

![image]({{ "/assets/img/posts/2020-01-17-image-10.png" | relative_url }})

You should now see the Microsoft Forms 2.0 Object Library in the list. Make sure it's checked and click OK.

![image]({{ "/assets/img/posts/2020-01-17-image-11.png" | relative_url }})

Save and Close the VBA Editor.

![image]({{ "/assets/img/posts/2020-01-17-image-12.png" | relative_url }})

Get back into the Customize the Ribbon dialog.

Choose Macros from the "Choose commands from" list.

Select `PERSONAL.XLSB!CopyAsCSV`

![image]({{ "/assets/img/posts/2020-01-17-image-13.png" | relative_url }})

Then, browse in the right-hand list to where you want to place the new ribbon/toolbar command. I keep my macros under the Developer tab so as to not mess up any of the other default tabs. You can also create new tabs, groups, etc. by right-clicking in the right-hand list.

Click the `Add >>` button between the two lists.

Give the new button a more useful name and a pretty icon by selecting it in the right-hand list and clicking the Rename... button below the list.

![image]({{ "/assets/img/posts/2020-01-17-image-14.png" | relative_url }})

Click OK to exit the Rename dialog, and then OK again to exit the Customize the Ribbon dialog. Your new button should now be visible in the location you chose.

![image]({{ "/assets/img/posts/2020-01-17-image-15.png" | relative_url }})

**Give it a shot!**

![image]({{ "/assets/img/posts/2020-01-17-image-16.png" | relative_url }})

_Also, when did "Search with Bing..." get added to Notepad..._

Sources:

[https://answers.microsoft.com/en-us/mac/forum/macoffice2011-macexcel/copy-cells-and-paste-them-as-csv/b8a05dd8-4605-408a-a1da-9842c767745f?msgId=1aa0503e-d369-4d6b-909f-c8679ee44e3c&tab=AllReplies&page=1](https://answers.microsoft.com/en-us/mac/forum/macoffice2011-macexcel/copy-cells-and-paste-them-as-csv/b8a05dd8-4605-408a-a1da-9842c767745f?msgId=1aa0503e-d369-4d6b-909f-c8679ee44e3c&tab=AllReplies&page=1)

[https://www.excelforum.com/excel-programming-vba-macros/353942-how-do-i-reference-dataobject.html](https://www.excelforum.com/excel-programming-vba-macros/353942-how-do-i-reference-dataobject.html)
