---
layout: default
---

## SolarWinds Compromise Hunting
###### A work in progress. All suggestions come from FireEye's technical reports as they develop.

### Check for a single host logging in to several accounts
```Powershell
Invoke-Command -ComputerName (Get-Content ./yourHostnames.txt) { Get-EventLog -LogName security -InstanceId 4624 | Select-Object @{Name="Account";Expression={ $_.ReplacementStrings[5]}} | Group-Object Account | Select Count,Name | Sort-Object Count -Descending }
```

### Check Shodan/other internet scraping 'service' for your hostnames
https://www.shodan.io/search?query=yourHostnameHere

### Check network traffic for SMB patterns that match a file replace, execute, and replace again
#### Bro Fields
- SMB::FILE_DELETE
- SMB::FILE_WRITE
- SMB::FILE_OPEN
- SMB::FILE_DELETE
- SMB::FILE_WRITE
