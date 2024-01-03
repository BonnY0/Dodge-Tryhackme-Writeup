# Dodge THM---writeup



https://tryhackme.com/room/dodge


/////////
//RECON//
/////////


We begin with nmap scan of our target IP

nmap -sV -sC -p- -O 10.10.199.242 -vv

nmap found 3  open ports : 22, 80, 443

![nmap1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/9a97d6c3-7ab9-4515-a40f-eec09ca0b7e1)



And nmap also found subdomains of dodge.thm domain, we need to add them all to /etc/hosts/ file

![etchosts](https://github.com/BonnY0/Dodge---writeup/assets/65781644/acdbd1f6-e918-44bc-8803-63cd70cca7ac)


Now we visit each site including both 80 and 443 (http/https) ports
Most of them will result in 403 error

![error403](https://github.com/BonnY0/Dodge---writeup/assets/65781644/e6cbfe8a-585e-42df-9e31-453521ecf20f)


Visiting https://dev.dodge.thm/ and https://netops-dev.dodge.thm/ we get 2 new responses

First one is phpinfo() where we find working directory /var/www/html_www/ , we need to keep this in mind so if there is some sort of  
vuln on other sites we know where to look at as a working directory

Second one has a title Firewall - Upload Logs

![firewalltitle](https://github.com/BonnY0/Dodge---writeup/assets/65781644/08335b60-5758-49a9-b444-0c193c3a3b24)

Blank page but checking out source code we find firewall.js script, opening it we can see that the script loads firewall10110.php via GET method
![sourcecode](https://github.com/BonnY0/Dodge---writeup/assets/65781644/63d06b26-afc0-4589-aab6-57ee04675505)
![sourcecode2](https://github.com/BonnY0/Dodge---writeup/assets/65781644/8eec9c6f-ed8f-4a5e-8a4f-5bb2cfda13a8)


Visitng https://netops-dev.dodge.thm/firewall10110.php
![firewallfoothold1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/b9b8afe9-ced6-4d79-927f-e93439ba70c3)



////////////
//FOOTHOLD//
////////////



We can see that there is a command field and firewall status 

Command and sql injection didnt work so a quick search around google  UFW Command

We find this site https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands

We find command that we can utilize 

![rules1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/43fc27a4-a9ab-4824-ad6b-ce79a1be7bcd)



But in our case we can see that there is port 21 with DENY IN so we input a command like 

sudo ufw allow 21

And get response 

Rule updated
Rule updated 


Refreshing the site we can see that port 21 has ALLOW IN set

So we re-run nmap 

nmap -p 21 10.10.199.242 -vv  

 //could also run -sC -sV and nmap will try to connect as anonymous but it takes longer to complete the scan

21/tcp open  ftp     syn-ack ttl 63

Now we attempt to connect to ftp as anonymous

![ftp1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/26fc3064-b251-4e53-bac6-f354c27dd429)


And navigate to .ssh dir

We can see that we can write and download authorized_keys and id_rsa_backup but not id_rsa

So we run mget * 

![ftp2](https://github.com/BonnY0/Dodge---writeup/assets/65781644/25f51884-2027-4e89-9f5c-3369efe93175)



And we check if id_rsa_backup has a password and it does not

![ssh1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/40ce949f-0147-4004-be3e-a744c684f1e0)



We check out authorized_keys to find possible user and we can see that there is user named challenger


![ssh2](https://github.com/BonnY0/Dodge---writeup/assets/65781644/4a70e1ec-a625-4340-ae2c-d7e4ee48c46c)


We need to set proper priv to id_rsa_backup

chmod 600 id_rsa_backup so we can ssh 

ssh -i id_rsa_backup challenger@10.10.199.242


and we get our first flag!


//////////////
//ESCALATION//
//////////////


We notice that there is .bash_history so we check it out

We can see that user cat out setup.php and posts.php but we dont know the location of those files

![esc1](https://github.com/BonnY0/Dodge---writeup/assets/65781644/c618d801-2583-431c-bed8-ad83b9968601)

locate doesnt work so we just run linPEAS , we can manually search for files

Running linPEAS could show us some stuff that would take a lot of time to find

linPEAS showed us /var/www/notes and .git files in /opt

Going to /var/www/notes we find api directory

![esc2](https://github.com/BonnY0/Dodge---writeup/assets/65781644/3f101c57-1b3d-43df-aea0-bf0adc28650c)

We can see the files posts.php and setup.php and config.php

We cant open setup.php but can posts.php and config.php

![esc3](https://github.com/BonnY0/Dodge---writeup/assets/65781644/ed059096-ab06-4a7b-8e65-df2d60e69571)

Inside posts.php we find base64 encoded string , we will utilize cyberchef to decode it

![esc4](https://github.com/BonnY0/Dodge---writeup/assets/65781644/9e2f3e02-926a-4a31-bdaa-cccbd8ce7172)

We find cobra user creds

su cobra

And run sudo -l 

![esc5](https://github.com/BonnY0/Dodge---writeup/assets/65781644/ebc0a48d-421b-4c28-8234-81c79c53cb6b)

Searching gtfobins for apt

![esc6](https://github.com/BonnY0/Dodge---writeup/assets/65781644/099654af-90c3-4534-9dd1-f70c27487073)

And running 

sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh

We get root#
