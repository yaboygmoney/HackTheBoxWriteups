---
layout: default
---

## Magnet Windows Forensics CTF 2022
##### Work in progress 

In this CTF, I'll be running Autopsy because I'm doing this a year after the event and I didn't get AXIOM.

Source file: https://storage.googleapis.com/mvs-2022/HP-Final.zip   
MD5: B28EE46B0CD62165D82AE41ED852D3BF

Challenge: Never Gonna Give You Up  
Points: 5  
How many times did Patrick get rick rolled?  
We can open up Data Artifacts > Web History in Autopsy and see that there were 9 times when Patrick opened the link to 'Never Gonna Give You Up' on YouTube.
![](https://yaboygmoney.github.io/htb/images/magnet22/q1.png)

Challenge: r/hobbies  
Points: 5  
What subreddit did Patrick frequent the most? Format: r/subredditName  
r/Stamps. Same logic as before: Data Artifacts > Web History, find reddit. 
![](https://yaboygmoney.github.io/htb/images/magnet22/q2.png)

Challenge: Version aversion  
Points: 5  
What was the version number of ZeroTier that was installed on the system?  
Data Artifacts > Installed Programs. ZeroTier One v.1.8.4.
![](https://yaboygmoney.github.io/htb/images/magnet22/q3.png)

Challenge: Insider Preview   
Points: 10  
What is the Build Number of the Windows Install  

Challenge: Nil Layer   
Points: 10  
What was the ZeroTier Network name?  
Check out local configurations for ZeroTier in Patrick's AppData Local. We can find "pensive_joybubbles" stored under "Name:" in the saved_networks.json file.  
![](https://yaboygmoney.github.io/htb/images/magnet22/q5.png)

Challenge: Crater of Diamonds   
Points: 10  
When did n30forever “Mine” diamonds? YYYY-MM-DD HH:MM UTC  

Challenge: Punching Wood  
Points: 10  
How many wood blocks has n30forever mined?  

Challenge: Default Skin  
Points: 25  
What is the SID of the account that was used to create the extra user?  
To answer this one, I exported the Security log from the disk image and loaded it into my local Event Viewer. A quick filter for Event ID 4720 (User Creation) had only one hit. The user minecraftsteve was created on 11 Feb 2022 at 7:29:43 PM. 
![](https://yaboygmoney.github.io/htb/images/magnet22/SID.png)
That user's SID is S-1-5-21-3341181097-1059518978-806882922-1002, which you might mistake for the answer at first. Event Viewer loses a lot of the context in the "General" tab. Moving to the "Details" tab, we can see the SubjectUserName & SubjectUserSid both point to the system. The TargetUserName is the user account that was created by the subject S-1-5-18.
![](https://yaboygmoney.github.io/htb/images/magnet22/SIDAnswer.png)

Challenge: Groundhog Day  
Points: 25  
Where did n30forever spawn on the most recent logon? Format: x,y,z  
If we go into the Minecraft logs in C:\Users\Patrick\Minecraft\logs and open the most recent log, towards the bottom of the log we can see a login at 226.5624, 71, 200. Bonus Log4jRCE.
![](https://yaboygmoney.github.io/htb/images/magnet22/loginLocation.png)

Challenge: Real 2020 Moment   
Points: 50  
Patrick reports seeing a couple of notifications saying malicious files were found and quarentined. What was the file name of the malicious file that was spawned with the process name starting with the letter S?  

Challenge: 1T5 H4CK1N6 T1M3   
Points: 50  
Patrick reports a suspicious console open on his screen. Can you find the full path to the script that caused this?  
For this one, we got a clue from the question. There are only so many types of scripts that can run on Windows, and the ones that generally pop a console are BAT files. In Autopsy, File Views > File Types > By Extension > Executable > .bat. There were only 34 here, but one was in a weird location. C:\Users\Patrick\lolololol\matrix.bat 
![](https://yaboygmoney.github.io/htb/images/magnet22/hackedBro.png)

Challenge: Philatelists Club   
Points: 50  
Patrick put a sign outside of his house in his offline survial minecraft world. What did it say? (Combine all lines of sign into 1 flag)  

Challenge: Time 2 Bock 0.0.0.0/0   
Points: 75  
What is the full address that the backdoor was downloaded into the system from? xxx.xxx.xxx.xxx:xxx/directory/backdoor.extension  
Earlier when looking through .txt files, I noticed one that pointed me towards the Minecraft folder in Patrick's documents. I also noticed a "goteem.ps1" file, so I started looking through the PowerShell logs. In there, I was able to find a local download for PowerCat.ps1. 
![](https://yaboygmoney.github.io/htb/images/magnet22/localDownload.png)

Challenge: Obfuscation Occasion   
Points: 75  
Locate and extract the file identified in the above question. What is the first function name in the malware? **Caution sample is REAL MALWARE**  

Challenge: Oh Boy Its Time to DCode   
Points: 100  
It looks like the vba file contains another encoded file. Decode this and provide the time/date stamp located inside the COFF header in UTC. yyyy/mm/dd:HH:MM:SS **Caution sample is REAL MALWARE**  
