---
layout: default
---
<html>
<div class="topnav">  
  <div style="float:right">
    <a href="https://yaboygmoney.github.io/htb/index.html">HOME</a>
    <a href="https://yaboygmoney.github.io/htb/about.html">ABOUT</a>
    <a href="https://yaboygmoney.github.io/htb/machines.html">MACHINES</a>
  </div>
</div>
</html>

# HackTheBox Writeups
### hours of effort summed up in a 3 minute read

## Heist
###### Retired 30 November 2019
![](https://www.hackthebox.eu/storage/avatars/131dbaba68b169bd5ff59ac09420b09f.png)

##### Intro
This was one of the most realistic HTB machines I've done so far. User required cracking several passwords that were found in router/switch configurations that bad network admins end up emailing around. Root was a refreshing break from the typical permissions abuse and exploited a common flaw I think we're all guilty of: password reuse.

I've already added the machine to my /etc/hosts file as heist.htb. Starting with nmap:

```bash
nmap -T5 -A -p 1-65535 -n -v -o nmap.results heist.htb
```
![](https://yaboygmoney.github.io/htb/images/heist/nmap.JPG)

Our nmap results tell us we've got a website on 80, SMB on 135/445, and Windows Remote Management on 5985. I start by poking around the website. We land on a login page.

![](https://yaboygmoney.github.io/htb/images/heist/loginpage.JPG)

Opting for the path of least resistance, I login as guest and am met with a helpdesk page.

![](https://yaboygmoney.github.io/htb/images/heist/guestlogin.JPG)

This page gives us a potential username: Hazard. Checking out the attachment Hazard included provides a big breakthrough.

![](https://yaboygmoney.github.io/htb/images/heist/config.JPG)

The config gives two additional usernames: rout3r and admin. Additionally, there are 3 password hashes present. The Type 7 passwords crack mad fast. I used an online cracker to save my CPU the slight stress. All we need is the password string

![](https://yaboygmoney.github.io/htb/images/heist/cisco7.JPG)
![](https://yaboygmoney.github.io/htb/images/heist/admincisco7.JPG)

Cracking the Type 5 was a little more involved. I fired up hashcat on my host machine and pasted the hash into a text file called hash.txt. Then I set up the hashcat command like so:

```bash
hashcat64.exe -m 500 hash.txt rockyou.txt
```

```-m 500``` specifies the type of hash for hashcat. 500 is a Cisco Type 5/MD5 hash. I pointed it against the famous rockyou password list and a few minutes later had a result:

![](https://yaboygmoney.github.io/htb/images/heist/enable5hashcat.JPG)

At this point, we have three sets of creds:

| Usernames | Passwords |
| --------- | --------- |
| admin | Q4)sJu\Y8zq\*A3?d |
| rout3r | $uperP@ssword |
| Hazard | stealth1agent |

The website login required an email, and I had no idea what the domain might be at this point so logging into the webpage would be a time sink.

I'm greedy right off the bat and I wanted WinRM capability. Google pointed me to Evil-WinRM. Evil-WinRM is a ruby tool that hooks you up with a fully functional PowerShell shell, complete with ```upload``` and ```download``` capabilities. 

I tried to run it with each of these usernames and passwords in various combinations to no avail. WinRM apparently isn't the direction I need to go just yet. I turn back to SMB. 

Based on the guest login from the web page, we know that Hazard had an account created for him on the server, so I login to SMB with the ```Hazard:stealth1agent``` combination and it's legit. I needed to enumerate more users for WinRM, so I launched the ```lookupsid.py``` tool I downloaded from Impacket. This tool brute forces SIDs on the server and finds both workgroups and usernames.

```bash
./lookupsid.py hazard:stealth1agent@heist.htb
```

![](https://yaboygmoney.github.io/htb/images/heist/smbUsers.JPG)

If I had a ridiculous list of usernames and passwords I would have scripted this next part out, but the list was relatively short so at this point I just start trying username and password combinations with Evil-WinRM. I skipped down to the SIDs starting at 1000+ since they're less likely to be built-in accounts. Eventually (I say that, it probably only took 5 minutes) I get to Chase and the Q4)sJu\Y8zq*A3?d password with the command:
```bash
evil-winrm -i heist.htb -u Chase -p "Q4)sJu\Y8zq*A3?d"
```

![](https://yaboygmoney.github.io/htb/images/heist/evilwinrm.JPG)

Evil-WinRM dropped me right into a shell. I've gotten Windows machines before, and knew the flags typically exist on the desktop, but if you didn't know where to go a quick way to find it would be:

```
cd C:\Users\Chase
Get-ChildItem -file -recurse | where-object -property name -eq user.txt
```

This little bit of PowerShell would look through all of Chase's folders and find the file named user.txt.

![](https://yaboygmoney.github.io/htb/images/heist/usermasked.jpg)

```type``` is the Windows CLI equivalent to ```cat``` for those unfamiliar. Alternatively, since this is a PowerShell shell, I could have also used the ```Get-Content``` cmdlet.

Now that I have user, I start looking for privesc. Going through a Windows privesc cheat sheet is the best way for to stay organized. There wasn't anything crazy installed in the C:\Program Files\ directory. I ran ```tasklist``` to see what was running but I was denied. ```Get-Process``` is the PowerShell equivalent, and that was permitted. There wasn't anything extraordinarily interesting. Firefox was pretty much the only non-OS process.

![](https://yaboygmoney.github.io/htb/images/heist/procs.JPG)

Browsers have the wonderful capability to store passwords, and some googling taught me they were encrypted in logins.json files that require keys3.db or keys4.db files to decrypt them. I found a key3.db file, but no logins.json file. Herein lies the flaws of write ups, they make it sound easy. I spent an inordinate amount of time (probably over an hour) trying to carve a password out of these files. I downloaded a .ja archive file thinking the .ja file might exist in there and messed with it for a minute. Eventually I decided to give up on this and moved on.

A tool called ```firefox_decrypt.py``` exists that can extract passwords out of Firefox profiles, but Python was not present on this machine, so that was a dead end. I started downloading the entire profile with the Evil-WinRM ```download``` command but it was going so painfully slow that I killed it before it finished.

Then I remembered that Firefox was a running process, loaded into memory. Task Manager provides a super easy "Dump" button to create a memory dump, but I didn't have a GUI. So on my Kali machine I went and downloaded procdump64.exe from SysInternals, and then in my Evil-WinRM shell I typed ```upload procdump64.exe```. 

There were 4 Firefox processes, so procdump needed a PID to know which one to dump. I ran:
```./procdump64.exe -accepteula -ma [Firefox PID]``` for a couple of the Firefox processes. 

![](https://yaboygmoney.github.io/htb/images/heist/memdump.JPG)

Firefox has a "Master Password", so I first looked for that. For those that don't Windows, ```findstr``` is Windows grep. ```/i``` ignores case and ```/C:``` lets you search a phrase. If you don't specify ```/C:```, findstr will find all instances of "master" AND "password" whether they appear together or not. That will pull back a LOT from this memory dump.
```
findstr /i /C:"Master Password" [memory dump filename]
```
That didn't pull anything useful back. Then I made the mistake of searching for "administrator" and "password" and that pulled back WAY too much. I went to the source code for the login page and got the parameter name for the password to help build a narrower search.

![](https://yaboygmoney.github.io/htb/images/heist/source.JPG)

The field was called "login_password". I ran another findstr command looking for our parameter name:
```
findstr "login_password" [memory dump filename]
```
![](https://yaboygmoney.github.io/htb/images/heist/dumped_LI.jpg)

That came back with a way more manageable result. Inside, we find the URL that was passed. In the url was creds for ```admin@support.htb``` with the password 4dD!5}x/re8]FBuZ.

I opened up a new terminal window and Evil-WinRM'd as admin with the password:
```bash
evil-winrm -i heist.htb -u administrator -p "4dD!5}x/re8]FBuZ"
```

![](https://yaboygmoney.github.io/htb/images/heist/root_LI.jpg)

Password reuse strikes again! Admin's webpanel login is the same as their WinRM password, providing me with a root shell and the root flag.

Overall, I was really impressed with this machine. The realism was there and it's great to get on a Windows machine. Most importantly, I learned a couple of new tools including lookupsid.py and Evil-WinRM. Evil-WinRM is a great shell and I'll definitely be using it again.

#### Keep trying
