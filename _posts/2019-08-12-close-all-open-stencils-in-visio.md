---
title: "Close all open stencils in Visio"
slug: "close-all-open-stencils-in-visio"
date: "2019-08-12T15:02:59"
categories: ["Uncategorized"]
tags: ["code", "visio"]
---
If you’re like me, you’ve probably got every stencil from every vendor always open in Visio. This macro code will close them all in one go.

```
Sub CloseAllStencils()

    'Enable diagram services
    Dim DiagramServices As Integer
    DiagramServices = ActiveDocument.DiagramServicesEnabled
    ActiveDocument.DiagramServicesEnabled = visServiceVersion140 + visServiceVersion150

    For Each doc In Application.Documents
        If doc.Type = visTypeStencil Then
            Debug.Print (doc.Name)
            doc.Close
        End If
    Next
    'Application.ActiveWindow.Windows.ItemEx("CCH-CP12-91").Close

    'Restore diagram services
    ActiveDocument.DiagramServicesEnabled = DiagramServices

End Sub
```
