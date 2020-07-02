---
layout: post
title:  "Stapler 1 Write-Up"
categories: [write-ups]
permalink: /write-ups/stapler_1_writeup/
---
FristiLeaks is an easy OSCP-like boot to root VM hosted on Vulnhub. The goal in this challenge is to obtain root in the box and read the flag.<br>
As stated in the Vulnhub page, there are minimum 2 ways to obtain a limited shell and 3 ways to obtain root.
<h1>Network Scan</h1>
In this case, the machine doesn't tell us its IP address, so I used Nmap to scan the network and find it.
![Network Scan Result](netScan.png)
I know that 192.168.56.1 is the gateway and 192.168.56.10 is the Kali machine, so 192.168.56.128 must be the target box.
<br><br>
<h1>Port Scan</h1>
The port scan reveals lots of open ports, including one which is unusally high. 
![Port scan result](allPortsScan.png)
<br>
<h1>FTP Anonymous Login</h1>
Port 21 has a ftp server with anonymous access configured, as shown in the Nmap output.
![FTP Nmap output](ftpNmap.png)
Just from connecting we get a possible user (Harry) and using user = anonymous and any password we get anonymous access, with allows use to read a file with two possible usernames (Elly and John).
![FTP Anon 1](ftpAnon1.png)
![FTP Anon 2](ftpAnon2.png)
{% highlight c %}
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
{% endhighlight %}
Reading other people write-ups I found that Elly uses the password ylle for her ftp account, but I used another method to get access.
<br><br>
<h1>HTTP Port 80</h1>
Here there isn't anything interesting, using gobuster we can find that it is serving a home directory but I didn't find anything useful.
![Root HTTP 80](root80.png)
![Gobuster HTTP 80](gobuster80.png)
<br>
<h1>Samba Enumeration</h1>
Port 139 has a Samba server which allows Guest login. There isn't anything interesting inside the shares, but using enum4linux we can enumerate the users.
![SMBMap](smbmap.png)
![enum4linux users](enum4linux.png)
<br>
<h1>Obtaining Credentials (Method 1)</h1>
At this point I got stucked because of the rabbit holes in the other ports (except 12380, but I will talk about this one in the second method). I tried using Hydra to bruteforce the ftp credentials, using the user list as input for both the user and the password.
![Hydra ftp](hydraFTP.png)
If we enter access the ftp server using these credentials we get access to the /etc/ directory. I searched for sensible information inside the directory, but I only found the /etc/password file which contains all the users inside of the box.
I used the same method as before to bruteforce the credentials for the ssh, only to find out that SHayslett uses the same password as before for his user account.
![Hydra ssh](hydraSSH.png)
<br>
<h1>Method 2</h1>
At first glance port 12380 doesn't have anything (except for another username), but if we inspect the responses using Burpsuite we can see that something is odd, the server always responds with a 400 Bad Request. (I don't know why the first screenshot doesn't display correctly, maybe it is too big).
![Root HTTP 12380 1](root12380_1.png)
![Comment with user](comment12380_1.png)
![Burp intercept](burpIntercept.png)
Using Nikto we can see that the server wants to use HTTPS, that's why we are getting a 400 response. Also, we can see that there is a robots.txt file with some disallowed entries.
![Nikto](nikto.png)
The /admin112233 is a troll page, while the /blogblog has a Wordpress blog.
![blogblog page](blogblog.png)
I used wpscan but it couldn't find anything, so I enumerated it manually.<br>
The wp-contents/plugins/ has directory listing enabled, so we can enumerate the plugins. Using searchsploit we can see that one of the plugins was vulnerable in its 1.0 version (I couldn't find the version of the installed plugin, but it seems to be 1.0).
![Wordpress plugins](wpplugins.png)
![Searchsploit plugin](searchsploitPlugin.png)
The searchsploit code didn't work correctly, but searching in Google I found a [more stable version](https://raw.githubusercontent.com/gtech/39646/master/39646.py) which was created for this box.
![Exploit](mysqlCreds.png)
In the output we can see the credentials for the root user of mysql. In the Nikto output we can see that there is a phpmyadmin page, so I used the newly obtained credentials and enumerated the database (which contained interesting information, but nothing useful).<br>
I used some SQL tricks to place a simple php shell inside the /wp-content/uploads/ directory and get RCE. (Ignore the typo in the screenshot).
![RCE 1](rce1.png)
![RCE 2](rce2.png)
![RCE 3](rce3.png)
![RCE 4](rce4.png)
At this point we can use a reverse shell to get shell access to the box, but I only used the first method because I prefer to have a more stable shell.<br>
Also, Zoe uses the same password as the root database user in the box, so we can get SSH access from there.
![Hydra ssh 2](hydraSSH2.png)
<br>
<h1>PrivEsc Method 1 (.bash_history)</h1>
Reading the LinEnum output we can see all the contents of the .bash_history of all users. JKanode has two passwords there.
![Bash history](bashhist.png)
Peter can run sudo as root.
![Sudo 1](sudo1.png)
![Sudo 2](sudo2.png)
And from here we can read the flag inside the /root/flag.txt file.
<br><br>
<h1>PrivEsc Method 2 (cronjob)</h1>
Reading the cron jobs we can see that root runs the /usr/local/sbin/cron-logrotate.sh script every 5 minutes and everyone has permission to write to this file.
![Cronjob](cronjob.png)
This can also be found using the find command.
![Find writable](findWrite.png)
We can just grant sudo privileges to the user that we are using (zoe in my case).
![Cron modified](cronModified.png)
![Zoe sudo](zoeSudo.png)
Just remember to remove the command from the cron-logrotate to avoid problems with the /etc/sudoers file.
<br><br>
<h1>PrivEsc Method 3 (Kernel exploit)</h1>
Again, reading the LinEnum output we can see the kernel version and the Ubuntu version. Using searchsploit we can see lots of exploits for this version, but most of them are for 64 bits operating systems and the one in the machine uses 32 bits.<br>
The exploit was a bit weird and I had to execute it two times before I got root.
![Kern exploit 1](kern1.png)
![Kern exploit 2](kern2.png)
![Kern exploit 3](kern3.png)
![Kern exploit 4](kern4.png)
<br>
<h1>Conclusion</h1>
This box has a lot of rabbit holes and can be pretty confusing if you don't take notes.<br>
I've learnt how to enumerate Samba shares and use enum4linux, as well as reminding myself to read more carefully the LinEnum output and search for writable files.
