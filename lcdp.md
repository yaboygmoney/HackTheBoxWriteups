---
layout: default
---

## LaCasaDePapel
###### Retired 27 July 2019
![](https://www.hackthebox.eu/storage/avatars/509c1d6ddf04cf3d3f8054a564f2e93a.png)

La Casa De Papel. Gaining an initial foothold was rough. I had never messed with the openssl command (which is massive) or forging client SSL/TLS certs before this. Pair that with a mega crashable HTTPS server and getting user was a trip. Fortunately, privesc wasn’t overly complicated.


As most do, we start with nmap.

```bash
nmap -sV -o nmap.results 10.10.10.131
```
![](https://yaboygmoney.github.io/htb/images/lcdp/nmapResults.JPG)

As a normal step, I ran gobuster in the background but I didn’t find anything of interest on port 80. There was a ‘secret token’ on the front page in the source but nothing I could work with. When we try to visit 443 it wants a client certificate, which we do not have. Dead end.

![](https://yaboygmoney.github.io/htb/images/lcdp/noClient.JPG)
 
Starting at the top of the nmap results list, FTP version vsftpd 2.3.4 is *old* and has an available exploit that when a user attempts to login with a smiley, it kicks open the server on port 6200 and enables unauthenticated access. A full breakdown of the vulnerability is available [here](https://www.hackingtutorials.org/metasploit-tutorials/exploiting-vsftpd-metasploitable/).
To exploit this vulnerability, we can use either telnet or netcat to first connect to port 21. We submit

```USER invalid:)```

as our username and 

```PASS idk```

for our password. The password can be literally anything. The function to open the secondary port is executed by the username smiley.

![](https://yaboygmoney.github.io/htb/images/lcdp/ftpHang.JPG)

The FTP server hangs after we submit those credentials. After that login function runs we can then connect a second time, this time to port 6200.

![](https://yaboygmoney.github.io/htb/images/lcdp/psyShell.JPG)

Connecting to the new port with netcat shows us a banner
‘Psy Shell v0.9.9 (PHP 7.2.10 – cli) by Justin Hileman’

As with a lot of specific technologies on HackTheBox, I’ve never encountered Psy Shell before. A quick Google tells me it’s a php debugging shell.
When in doubt, run ```help```

![](https://yaboygmoney.github.io/htb/images/lcdp/help.JPG)

Awesome. First things first, I run ```ls``` to see what’s going on.

![](https://yaboygmoney.github.io/htb/images/lcdp/ls.JPG)

Looks like there’s a single variable here. I ran ‘help ls’ and saw –constants to show all of the constants. I ran it out of curiosity.  Don’t do that.

![](https://yaboygmoney.github.io/htb/images/lcdp/constantsMess.JPG)

The next command I run help against is ‘show’ like ```help show```

![](https://yaboygmoney.github.io/htb/images/lcdp/help%20show.JPG)

The example ```show $myObject``` would probably work pretty well with ```show $toyko``` from our ```ls``` results from earlier.

![](https://yaboygmoney.github.io/htb/images/lcdp/methodPrint.JPG)

This is a function that exists to sign an SSL client cert with the CA Root Key. I just didn’t know how to get it. Turns out, to get that CA root key you just need to call the function within the Psy Shell.

```file_get_contents('/home/nairobi/ca.key');``` pays off.

![](https://yaboygmoney.github.io/htb/images/lcdp/CARootKey.JPG)

Now, I could hella fancy and fix this key with sed and all. But I’m lazy and opening it with ```gedit``` and doing a find and replace is just as fast. Copy the text, paste it into gedit, find and replace the “\n” with nothing, and get rid of the spaces at the beginning of each line and we’re in business with a fully functional CA root key. Now we can use this CA root key to make our client cert for the 443 webpage.

Disclaimer: The protégé Alec ([imthebest](https://www.hackthebox.eu/home/users/profile/105191)) figured this next cert bit out. I messed with it for about an hour and decided to do something else. This dude sat there for a long time reading documentation, got it to work, and then so graciously taught me.

Step 1: We need to create a private key of our own, as well as a certificate signing request (csr).
```bash
openssl req -newkey rsa:2048 -nodes -keyout mykey.key -out lcdp.csr
```
Mykey.key is obviously my new private key, and lcdp.csr is the request that we will be signing with our newly stolen root key in just a few steps.

When making the csr, I learned that you can’t just put a dot in every field, hence the ‘a’ in Common Name. It doesn’t really matter what you put in here.

![](https://yaboygmoney.github.io/htb/images/lcdp/step1.JPG)

Aside from the CSR and CA root key, we need a third ingredient: the cert from the website. You could curl it to get it. I just used Firefox to export it. Click on the little padlock with the alert symbol next to the URL > View Certificate > Details > Export. Save that dude to your working directory. I saved mine as the default “lacasadepapel.crt”.

![](https://yaboygmoney.github.io/htb/images/lcdp/siteCert.JPG)

Step 2: Take your CSR, the cert we just exported, and the CA root key and mash all of those up into a client cert.
```bash
openssl x509 -req -in lcdp.csr -CA lacasadepapelhtb.crt -CAKey CA.key -CAcreateserial -out my.crt -days 1825
```
![](https://yaboygmoney.github.io/htb/images/lcdp/step2.JPG)

Let’s break this behemoth down (```openssl``` is a monster of a command, btw). ```-in lcdp.csr``` says “here’s my request”. ```-CA``` says “here’s the site’s cert”. ```-CAKey``` says “here’s the Certificate Authorities private key”. ```-out my.crt``` is what we want to name our new magic client cert. ```-days``` is how many days the cert is valid for.

Step 3: Merge your private key and the client cert into a pkcs12 file so that Firefox will use it.
```bash
openssl pkcs12 -inkey mykey.key -in my.crt -export -out lcdp.pfx
```
Then we open up Firefox and add our new pfx/pkcs12 file.
Firefox > Preferences > Privacy & Security > View Certificates > Your Certificates > Import

![](https://yaboygmoney.github.io/htb/images/lcdp/yourCerts.JPG)

Once we refresh the page on 443, we get this.

![](https://yaboygmoney.github.io/htb/images/lcdp/thissitehas.JPG)

Dooope. Say ok.
Now the webpage looks a little different. We get into the ‘Private Area’.

![](https://yaboygmoney.github.io/htb/images/lcdp/private%20area.JPG)

Just clicking around shows us that SEASON-1 is fetched with the URL of https://10.10.10.131/?path=SEASON-1. My brain immediately gets the idea of local file inclusion (LFI).

After clicking on Season-1, we’re met with a lot of AVI files. They don’t actually do anything, but we can download them. The download URL is interesting. It’s ```http://10.10.10.131/file/``` + a random string of characters, but for each file the string starts with ```U0VBU09OLTEvMD```.

![](https://yaboygmoney.github.io/htb/images/lcdp/links.png)

Interesting. I copy the string and pipe it into base64 -d.

```bash
echo U0VBU09OLTEvMD | base64 -d
```

![](https://yaboygmoney.github.io/htb/images/lcdp/decode.JPG)

Turns out our string is the filepath base64’d. Naturally, I try ```../../../../../../../etc/passwd``` to see if we can’t get away with something like that.

![](https://yaboygmoney.github.io/htb/images/lcdp/etcpasswd.JPG)

Spoiler: we can’t.

![](https://yaboygmoney.github.io/htb/images/lcdp/crushed443.JPG)

A bad GET request actually crushes HTTPS for us for a while, so it’s best to be cautious with what we request to save our sanity and time. Something else we can try is to just find /home. I navigate to https://10.10.10.131/?path=../../../../../../home and get something useful: users.

![](https://yaboygmoney.github.io/htb/images/lcdp/home.JPG)

I pass this URL and my cert to ZAP and let it spider the page. Berlin’s ssh private key “id_rsa” is sitting in ```/home/berlin/.ssh/```. Unfortunately the link isn't clickable, it's only text.

![](https://yaboygmoney.github.io/htb/images/lcdp/privKey.JPG)

If only we could download it.

What if we base64 encoded the path and sent that as a follow on to the ```https://10.10.10.131/file/``` URL from earlier?
Once again, I am lazy. I open [CyberChef](https://gchq.github.io/CyberChef/) and type in my string and add “To Base64” to my recipe.

![](https://yaboygmoney.github.io/htb/images/lcdp/chef.JPG)

Now we should be able to tack on that output to the /file/ URL and get the key.
https://10.10.10.131/file/\[ourbase64string]

![](https://yaboygmoney.github.io/htb/images/lcdp/getKey.JPG)

Got it. Might as well get the user flag too.

![](https://yaboygmoney.github.io/htb/images/lcdp/user.JPG)

Okay. Back to the SSH key we just comp’d. chmod the file so it’s hella restricted or else it will be ignored. See:

![](https://yaboygmoney.github.io/htb/images/lcdp/badperm.JPG)

```bash
chmod 600 id_rsa
```

Now we can SSH into the box as berlin using the private key and (hopefully) no other credentials will be required. To do that we use ```-i``` for identity file.
```bash
ssh berlin@10.10.10.131 -i id_rsa
```
![](https://yaboygmoney.github.io/htb/images/lcdp/password.JPG)

Remember when I put "hopefully" in parenthesis? Applicable. It asking for a password means that either this key still requires a password, or it's not valid for this user.

We have no other information, so I guess we try other users. It’s a private key we stole from this machine. It’s not useless (again, hopefully). Back when we LFI’d to the /home directory we saw berlin, dali, nairobi, oslo, and professor as users on the machine. It’s always the last one you try.

![](https://yaboygmoney.github.io/htb/images/lcdp/sshd.JPG)

We’ve landed on the box as the user professor. A quick directory listing shows we have a few files in our home directory.

![](https://yaboygmoney.github.io/htb/images/lcdp/SGID%20set.JPG)

This directory listing is very useful to us. First, we see that the SGID bit is set on the directory itself. This will allow us to inherit the permissions of whatever the group owner is. Next, we see that the owner of the file ‘memcached.ini’ is root. Any ini, conf, or config file we can write to and have run as root is money. If we ```cat``` those out:

![](https://yaboygmoney.github.io/htb/images/lcdp/memcached.JPG)

It looks like the ini file is setting a command variable that we can only assume will be executed once ```/usr/bin/memcached``` is ran. We're going to try and exploit that permission.

This next screenshot has quite a bit going on so we’ll step through. Before we start, notice I didn’t completely obliterate the original memcached.ini file. Backups, homies. Other people play here, too. 

![](https://yaboygmoney.github.io/htb/images/lcdp/steps.JPG)

I made a copy of the OG memcached.ini as memcached.ini.bak in the first ```cp``` command. Next, I created a file called ‘new’ and opened it with ```vi new```. Once in vi, I pasted the format of the original memcached.ini but replaced the command with a python reverse shell. 

In another terminal, I setup my netcat listener with 
```bash
nc -nvlp 1234
````

Finally, I renamed ‘new’ to ‘memcached.ini’.

After we get the new ini file written, we wait for root's cron job to run the binary.

Switching tabs, we can see that we caught our root shell.

![](https://yaboygmoney.github.io/htb/images/lcdp/rooted.JPG)

All that’s left to do is tidy up. Copy the original memcached.ini file back to its original place, delete the backups, and submit the flag.

Again, special thanks to Alec (sarcastically named imthebest on HTB) for grinding through those ```openssl``` commands. To us filthy casuals, cert forgery was no joke.

Until next time.
