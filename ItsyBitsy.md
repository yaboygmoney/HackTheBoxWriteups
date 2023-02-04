---
layout: default
---

## ItsyBitsy
###### Put your ELK knowledge together and investigate an incident.

Here's the scoop: During normal SOC monitoring, Analyst John observed an alert on an IDS solution indicating a potential C2 communication from a user Browne from the HR department. A suspicious file was accessed containing a malicious pattern THM:{ ________ }. A week-long HTTP connection logs have been pulled to investigate. Due to limited resources, only the connection logs could be pulled out and are ingested into the connection_logs index in Kibana.



1. How many events were in March 2022? 1482. Just change the date range in the discover tab.
2. What is the IP associated with the suspected user in the logs? 192.166.65.54. There are only two source IPs in the whole index and only one IP address goes to pastebin.
3. The user’s machine used a legit windows binary to download a file from the C2 server. What is the name of the binary? bitsadmin. Windows is really cool and makes that the user agent for us.
4. The infected machine connected with a famous filesharing site in this period, which also acts as a C2 server used by the malware authors to communicate. What is the name of the filesharing site? We already said pastebin.
5. What is the full URL of the C2 to which the infected host is connected? pastebin[.]com/yTg0Ah6a
6. A file was accessed on the filesharing site. What is the name of the file accessed? I hate this question for teaching defensive ops because I feel like it could be training operators/analysts to go interact directly with C2 sites rather than using other datasources to figure things out. When we find a breach, we don't want to immediately show our hand that we've detected it in the event the adversary is monitoring for new C2 connections. But, to accomplish this one since we only have connection logs and not data we have to go the pastebin site. There, you'll find secret.txt.

Here's the only document you need to solve this room

![](https://yaboygmoney.github.io/htb/images/log.png)
