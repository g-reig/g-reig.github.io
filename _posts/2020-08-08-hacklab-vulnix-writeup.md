---
layout: post
title:  "HackLab: Vulnix Write-Up"
categories: [write-ups]
permalink: /write-ups/hacklab_vulnix_writeup/
---
HackLab: Vulnix is an intermediate rated OSCP-like machine hosted on Vulnhub (and the last rated machine on Abatchy's list).
<h1>Network Scan</h1>
As always, I've used Nmap to sweep the network and get the IP of the target machine.
![NetScan](netScan.png)
The IP of the target machine is 192.168.56.133.
<br><br>
<h1>Port Scan</h1>
![PortScan](portScan.png)
There are lots of open ports in this machine and most of them lead to anything. I'll explain what can be obtained in some of them, but I don't know how to deal with all of them.
<br><br>
<h1>Ports Explanation</h1>
In port 25 there is a SMTP server which has the VRFY command enabled. This can be used to enumerate some users using dictionaries and to send malicious mails to them (as the server doesn't have any type of authentication). I'll show this in my writeup for SneakyMailer from HackTheBox when it gets retired.<br>
![Mail Enumeration](mailEnum.png)
Port 79 has a Finger server, which can also be used to enumerate users in the machine. In this case we can get more information about them like the home directory, shell used... In earlier versions of it the server could be exploited to gain shell access to the machine.
![Finger Enumeration](fingerEnum.png)
Pop3 and IMAP are protocols used to retrieve emails (the ones with an S at the end use encryption). They can be used to read the emails of the users if we get their credentials but in this case it isn't useful (Again, I'll show this in the writeup for SneakyMailer).<br><br>
RSH is a protocol similar to SSH but without encryption. RSH, Finger and SMTP (sendmail) had vulnerabilities in the past that were exploited by the Morris worm, so I suppose they are running in the machine as an "Easter Egg".<br><br>
I don't really know what to do with the higher ports, some of them are for RPC stuff but I don't know if they can be used to get more information.
<br><br>
<h1>NFS Enumeration</h1>
I began by running "showmount" against the machine to see what directories it was exporting. It showed the directory "/home/vulnix", which if you recall from before is the home directory for user vulnix.
![Showmount](showmount.png)
I mounted the directory using NFSv4 but I couldn't access it because I didn't had the privileges to do so. I also saw that the owner of the directory was nobody, which means that the root_squash flag was enabled (more on that later).
![Mounting NFSv4](nfsv4.png)
![NFSv4 Permissions](nfsv4Permissions.png)
NFS uses the UID and the GID to authenticate the user. The UID and GID of the user in the local machine must match with the ones in the folder that is being shared to access it. NFSv4 doesn't give out the ids for security reasons (even though they can be bruteforced with a lot of patience), but the same cannot be said for earlier versions, so I tried to mount the directory using NFSv3.
![Mounting NFSv3](nfsv3.png)
Here we can see that both the UID and the GID of the folder are 2008, so I created a user in my machine that matched these values. 
![User Creation](addUser.png)
I changed to the new user using the su command and I got access to the directory.
![Directory List](directoryList.png)
<br>
<h1>Alternative Method</h1>
Reading other people writeups I found that the machine has a user called "user" which uses a weak password. Running a bruteforce attack against the SSH server (or RSH) we can get his password. From there we can get the UID and GID of the owner and the group of the folder.<br>
I didn't use this method because the bruteforcing was going really slow and I was trying to bruteforce the credentials for root, user and vulnix, so I ended up gaining access with the first method before the bruteforce ended.
![Method 2](userMethod.png)
<br>
<h1>Gaining SSH Access</h1>
By default SSH uses the file inside /home/$USER/.ssh/authorized_keys to use public/private key authentication, so I added a key that I generated to try to get access to the machine without using the password.
![Foothold](foothold.png)
<br>
<h1>PrivEsc to Root</h1>
Running sudo -l we can see that vulnix can use sudoedit without supplying the password to edit the /etc/exports file, which is the file that defines the directories that are going to be exported by NFS.
![sudo -l](sudo-l.png)
I added the /root directory to the exports file with the "no_root_squash" flag enabled, which means that if we use the root user in our machine we can work as root in the directory shared with NFS (root_squash means that NFS ignores the requests made by root). 
![New exports](newExports.png)
To use the changes made in the exports file we need to restart the NFS service but we don't have permission to do so and I got stuck here for a long time. Eventually I gave up and looked for some hints on the Internet, just to find that everyone restarted the VM to restart the service. I did so and I got access to the /root directory with root privileges. I knew that root login was enabled in the /etc/sshd_config file, so I used the same method as before to get SSH access.
![SSH Config](SSHConfig.png)
![Showmount 2](showmount2.png)
![Mount root](mountRoot.png)
![PrivEsc](privesc.png)
<br>
<h1>Conclusion</h1>
This is a fairly difficult machine due to the amount of rabbit holes it has and the fact that you need to know a bit about NFS (and do some research on it) to get root (and user if you are too impatient). I didn't like the part about restarting the VM because it isn't something that you can do on a remote machine but overall this has been a really good learning experience.
