---
layout: default
---

## UnderTheWire - Century

When I first started learning how to use the Linux CLI, I learned by jumping right in [OverTheWire's Bandit](https://overthewire.org/wargames/bandit/). I recently came across a PowerShell version called [UnderTheWire](https://underthewire.tech/wargames.htm). This is a writeup covering the entirety of the [Century](https://underthewire.tech/century/century.htm) challenge. I'm skipping Century 0. The credentials are found in the Slack channel.

### Century 1
---
The password for Century2 is the build version of the instance of PowerShell installed on this system. 

```PowerShell
$PSVersionTable
```

Build Version is 10.0.14393.3383

### Century 2
---
The password for Century3 is the name of the built-in cmdlet that performs the wget like function within PowerShell PLUS the name of the file on the desktop.

```powershell
Invoke-WebRequest
Get-ChildItem -Path C:\users\century2\desktop | Select Name
```

### Century 3
---
The password for Century4 is the number of files on the desktop. 

```powershell
(Get-ChildItem ..\Desktop -file).count
```

### Century 4
---
The password for Century4 is the number of files on the desktop. 

```powershell
(Get-ChildItem ..\Desktop -file).count
```

#### Keep trying
