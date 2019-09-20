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

## Luke
###### Retired sometime in September? I've been busy.
![](https://yaboygmoney.github.io/htb/images/luke/machineImage.jpg)

##### Intro
Luke wasn't a difficult box, but it was all about organization. By the end, we had 4 different login panels, 5 sets of credentials, and lots of combinations to try.

First step is always our scan.
```bash
nmap -sV -n -oA nmap 10.10.10.137
```
![](https://yaboygmoney.github.io/htb/images/luke/nmap.jpg)

Our scan shows we have FTP, Web, NodeJS, and HTTP-Like on 8000. I just worked down the list here. 
FTP didn’t provide a whole lot. We get one text file, for_Chihiro.txt:

![](https://yaboygmoney.github.io/htb/images/luke/for_Chihiro.JPG)

This doesn't give us much, except two names we might be able to use later on, Chihiro and Derry, as well as their respective responsibilities. Moving on to the webpage.

![](https://yaboygmoney.github.io/htb/images/luke/index.JPG)

Not a ton going on. No forms to fiddle with. Good time to run gobuster. 
```bash
gobuster -x php,html -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -o gobuster.results -u 10.10.10.137
```

Gobuster gives us a few options to check out:

![](https://yaboygmoney.github.io/htb/images/luke/gobusterResults.JPG)

We're given the index page we've already visited, a login.php page, and a config.php page. 

I hit the config page first and it pays off:

![](https://yaboygmoney.github.io/htb/images/luke/creds.JPG)

We have a pair of creds now:
```
User: root
Password: Zk6heYCyv6ZE9Xcg
```

I confidently take those creds to the login.php page:

![](https://yaboygmoney.github.io/htb/images/luke/loginphp.JPG)

...and get nowhere. I try some typical web exploit stuff and nothing works, so I decide to move on and check out other ports.
Our nmap results showed an ```Ajenti http control panel``` so I take my creds there.

![](https://yaboygmoney.github.io/htb/images/luke/panellogin.JPG)

I try our creds there. Nope. I find the default creds for Ajenti (root:admin). Nope. I try some other usernames like administrator, luke, vader, darthvader (I tried a theme, ok?). 
No dice. Looking back at our list of ports my current situation is:

| Port | Status             |
|------|--------------------|
|  21  |  Exhausted         |
| 80   | Probably Exhausted |
| 3000 | Untouched          |
| 8000 | Probably Exhausted |

I decide to send gobuster after port 8000 and move onto node.js. I'm confident that this is where I'll use the database creds I found earlier.

I send a curl at the port and get a response:
```bash
curl -u root:Zk6heYCyv6ZE9Xcg http://10.10.10.137:3000
```

![](https://yaboygmoney.github.io/htb/images/luke/authTokenNotSupplied.JPG)

Okay, so that didn’t work but also didnt' totally fail. I have creds, but no token. Time to Google.
Fortunately I found [this](https://medium.com/dev-bits/a-guide-for-adding-jwt-token-based-authentication-to-your-single-page-nodejs-applications-c403f7cf04f4) (thanks Naren) that showed me how to get an auth token for JWT (JSON Web Tokens). That command ended up looking like this:
```bash
curl --header "Content-Type: application/json" --request POST --data '{"password":"Zk6heYCyv6ZE9Xcg", "username":"root"}' http://10.10.10.137:3000/login
```
All I got back was “Forbidden”. Come on. I used the creds you gave me. At this point, it’s all we have. Try other names, I guess? Derry? Chihiro? Nope. Admin? 

![](https://yaboygmoney.github.io/htb/images/luke/authenticated.JPG)

Bingpot. While we were working on this guy, gobuster found else something we might be interested in. http://luke.htb:3000/users. Cool cool cool cool cool.

Now we use the second command provided by Naren and stuff our token in there. That command looks like:
```bash
curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYyODAwODI2LCJleHAiOjE1NjI4ODcyMjZ9.9j55kMh03TkyPjiMoH97GOBbzaZedcY3YMg2C9cdfnY' http://10.10.10.137:3000/users
```

That gives us some good feedback:

![](https://yaboygmoney.github.io/htb/images/luke/userslist.JPG)

We get a list of usernames: Admin, Derry, Yuri, and Dory. If we poke just a bit deeper with our curl command and tack on a username after the /users/ URI, we can see a bit more information about each user.

![](https://yaboygmoney.github.io/htb/images/luke/curlPassword.JPG)

Now we have 5 total sets of creds. Our root set from the config.php page, plus:
```
Admin: WX5b7)>/rp$U)FW
Derry: rZ86wwLvx7jUxtch
Yuri: bet@tester87
Dory: 5y:!xa=ybfe)/QD
```

I take the admin set over to the port 8000 server webmin panel. Denied.
Same for Derry, Yuri, and Dory.

I try them out at login.php. Nope. I try these creds in various capitalization combinations in several locations and get no love.

I decide to blast dirb at port 80 this point in place of gobuster. Variety of tools is something that HTB has taught me thus far. Dirb finds something that gobuster did not, http://luke.htb/management. The reason that gobuster did not find it is because gobuster's default configuration ignores 401 replies while dirb does not. We hit up the management page and get met with a basic auth prompt.

![](https://yaboygmoney.github.io/htb/images/luke/managementLogin.JPG)

Feed it Admin creds and..nope. But wait. Think back to the FTP server. Derry is the frontend dev. Derry’s creds get us in to this:

![](https://yaboygmoney.github.io/htb/images/luke/managment.JPG)

Config.json is what we’re after here, as it has yet another set of creds.

![](https://yaboygmoney.github.io/htb/images/luke/configjson.JPG)

```
User: root
Password: KpMasng6S5EtTy9Z
```

I figuratively race to port 8000’s webmin panel and plug those babies in and gain access to the Ajenti panel:

![](https://yaboygmoney.github.io/htb/images/luke/ajentiauthd.JPG)

From here, it’s all over. I see ‘Terminal’ on the sidebar. I click ‘New’ to fire up a web shell and viola!

![](https://yaboygmoney.github.io/htb/images/luke/flag_LI.jpg)

I stopped here because I got my points, but if you’re a completionist, calling back a shell from here is trivial. Thanks for reading!

Until next time.

#### Keep trying
