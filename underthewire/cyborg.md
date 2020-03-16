---
layout: default
---

## UnderTheWire - Cyborg

When I first started learning how to use the Linux CLI, I learned by jumping right in [OverTheWire's Bandit](https://overthewire.org/wargames/bandit/). I recently came across a PowerShell version of these Wargames called [UnderTheWire](https://underthewire.tech/wargames.htm). This is a writeup covering the entirety of the [Cyborg](https://underthewire.tech/cyborg/cyborg.htm) challenge. The credentials for the first level are found in the Slack channel.

### Cyborg 1->2
---
Challenge: The password for cyborg2 is the state that the user Chris Rogers is from as stated within Active Directory. 

```PowerShell
Get-ADUser -filter 'Name -like "Rogers*"' -Properties state
```

### Cyborg 2->3
---
Challenge: The password for cyborg3 is the host A record IP address for CYBORG718W100N PLUS the name of the file on the desktop. 

```powershell
Resolve-DNSName -name CYBORG718W100N | select IPAddress
Get-ChildItem -Path C:\users\cyborg2\desktop | Select Name
```

### Cyborg 3->4
---
Challenge: The password for cyborg4 is the number of users in the Cyborg group within Active Directory PLUS the name of the file on the desktop. 

```powershell
(Get-ADGroupMember -Identity Cyborg).count
Get-ChildItem -file ..\Desktop
```

### Cyborg 4->5
---
Challenge: The password for cyborg5 is the PowerShell module name with a version number of 8.9.8.9 PLUS the name of the file on the desktop. 

```powershell
Get-Module -ListAvailable | where version -eq 8.9.8.9
Get-ChildItem -file ..\Desktop
```

### Cyborg 5->6
---
Challenge: The password for cyborg6 is the last name of the user who has logon hours set on their account PLUS the name of the file on the desktop. 

```powershell
Get-ADUser -Filter * -Property LogonHours | where logonhours
Get-ChildItem -file ..\Desktop
```

### Cyborg 6->7
---
Challenge: The password for cyborg7 is the decoded text of the string within the file on the desktop. 

```powershell
[System.Text.Encoding]::UTF8.Getstring([System.Convert]::FromBase64String((Get-Content ..\Desktop\cypher.txt)))
```

### Cyborg 7->8
---
Challenge: The password for cyborg8 is the executable name of a program that will start automatically when cyborg7 logs in. 

```powershell
Get-CimInstance win32_startupcommand
```

### Cyborg 8->9
---
Challenge: The password for cyborg9 is the Internet zone that the picture on the desktop was downloaded from. 

```powershell
Get-Content .\1_qs5nwlcl7f_-SwNlQvOrAw.png -stream Zone.Identifier
```

### Cyborg 9->10
---
Challenge: The password for cyborg10 is the first name of the user with the phone number of 876-5309 listed in Active Directory PLUS the name of the file on the desktop.  

```powershell
Get-ADUser -Properties telephonenumber -filter {telephonenumber -like '876-5309'}
Get-ChildItem -file ..\Desktop
```

### Cyborg 10->11
---
Challenge: The password for cyborg11 is the description of the Applocker Executable deny policy for ill_be_back.exe PLUS the name of the file on the desktop. 

```powershell
Get-AppLockerPolicy -Effective -xml
```

### Cyborg 11->12
---
Challenge: The password for cyborg12 is located in the IIS log. The password is not Mozilla or Opera. 

```powershell
Get-Content C:\inetpub\logs\logfiles\w3svc1\u_ex160413.log | select-string "pass"
Get-ChildItem -file ..\Desktop
```

### Cyborg 12->13
---
Challenge: The password for cyborg13 is the first four characters of the base64 encoded fullpath to the file that started the i_heart_robots service PLUS the name of the file on the desktop. 

```powershell
Get-WMIObject win32_service | where name -like i_heart_robots | select pathname
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("c:\windows\system32\cmd.exe")))
Get-ChildItem -file ..\Desktop
```

### Cyborg 13->14
---
Challenge: The password cyborg14 is the number of days the refresh interval is set to for DNS aging for the underthewire.tech zone PLUS the name of the file on the desktop. 

```powershell
Get-DNSServerZoneAging underthewire.tech
Get-ChildItem -file ..\Desktop
```

### Cyborg 14->15
---
Challenge: The password for cyborg15 is the caption for the DCOM application setting for application ID {59B8AFA0-229E-46D9-B980-DDA2C817EC7E} PLUS the name of the file on the desktop. 

```powershell
Get-WmiObject -class "win32_dcomapplication" | where appid -eq "{59B8AFA0-229E-46D9-B980-DDA2C817EC7E}" | select caption
Get-ChildItem -file ..\Desktop
```

That's it for Cyborg
#### Keep trying
