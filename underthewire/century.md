---
layout: default
---

## UnderTheWire - Century

When I first started learning how to use the Linux CLI, I learned by jumping right in [OverTheWire's Bandit](https://overthewire.org/wargames/bandit/). I recently came across a PowerShell version called [UnderTheWire](https://underthewire.tech/wargames.htm). This is a writeup covering the entirety of the [Century](https://underthewire.tech/century/century.htm) challenge. I'm skipping Century 0. The credentials are found in the Slack channel.

### Century 1->2
---
Challenge: The password for Century2 is the build version of the instance of PowerShell installed on this system. 

```PowerShell
$PSVersionTable
```

### Century 2->3
---
Challenge: The password for Century3 is the name of the built-in cmdlet that performs the wget like function within PowerShell PLUS the name of the file on the desktop.

```powershell
Invoke-WebRequest
Get-ChildItem -Path C:\users\century2\desktop | Select Name
```

### Century 3->4
---
Challenge: The password for Century4 is the number of files on the desktop. 

```powershell
(Get-ChildItem ..\Desktop -file).count
```

### Century 4->5
---
Challenge: The password for Century5 is the name of the file within a directory on the desktop that has spaces in its name. 

```powershell
Get-ChildItem ..\Desktop -recurse | Select Name
```

### Century 5->6
---
Challenge: The password for Century6 is the short name of the domain in which this system resides in PLUS the name of the file on the desktop.  

```powershell
Get-ADDomain | Select Name
Get-ChildItem -Path C:\users\century2\desktop | Select Name
```

### Century 6->7
---
Challenge: The password for Century7 is the number of folders on the desktop.  

```powershell
(Get-ChildItem C:\users\century6\desktop -Directory).count
```

### Century 7->8
---
Challenge: The password for Century8 is in a readme file somewhere within the contacts, desktop, documents, downloads, favorites, music, or videos folder in the user's profile. 

```powershell
Get-ChildItem C:\users\century7 -file -recurse | Where -property name -like *readme* | Get-Content
```

### Century 8->9
---
Challenge: The password for Century9 is the number of unique entries within the file on the desktop. 

```powershell
(Get-Content .\unique.txt | select -unique).count
```

### Century 9->10
---
Challenge: The password for Century10 is the 161st word within the file on the desktop. 

```powershell
$words = (Get-Content .\Word_File.txt).Split()
$words[160]
```

### Century 10->11
---
Challenge: The password for Century11 is the 10th and 8th word of the Windows Update service description combined PLUS the name of the file on the desktop. 

```powershell
Get-WMIObject win32_service | where -Property displayname -eq "Windows Update" | select description
Get-ChildItem ..\Desktop
```

#### Keep trying
