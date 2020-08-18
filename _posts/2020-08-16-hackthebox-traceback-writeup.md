---
layout: post
title: "HackTheBox: Traceback Write-Up"
categories: [write-ups]
permalink: /write-ups/htb_traceback_writeup/
---

Traceback is an easy rated machine hosted in HackTheBox. As always, the goal is to obtain root and read the flag in /root/root.txt
<h1>Port Scan</h1>
The IP is posted in the HackTheBox webpage, so it isn't necessary to do a network scan.
![Port Scan](port_scan.png)
From the port scan we can see that there are only two ports open (22 and 80).
<br><br>
<h1>WebPage Enumeration</h1>
The root page contains a message telling us that the website has been owned. If we read the source code we can see a little more information.
![Web SRC](web_src.png)
Searching the name "Xh4H" in Google we can find a github page with some repositories. One of them has the same text that was in the commentary and some webshells.
![WebShells](webshells.png)
<br>
<h1>WebShell Fuzzing</h1>
I cloned the repository in my machine and created a wordlist with the shells inside of it.
![WebShells Wordlist 1](webshells_wordlist1.png)
![WebShells Wordlist 2](webshells_wordlist2.png)
After this I used Wfuzz to fuzz the website using the wordlist to find the webshell.
![webfuzzing](webfuzzing.png)
<br>
<h1>Using the WebShell</h1>
Reading the source code for the webshell we can see the default credentials.
![Default Credentials](def_creds.png)
If we go to http://10.10.10.181/smevk.php we can see that the webshell has a login page.
![Login page](login_page.png)
The webshell has a console incorporated but I prefer to have a "normal" console so I edited a reverse shell that already was on the machine to connect to my machine.
![Files Web](files_web.png)
![Edit Reverse Shell](editing_rev.png)
![Connection](connection.png)
<br>
<h1>PrivEsc to Sysadmin</h1>
There is a text file inside the webadmin home.
![Note.txt](notes_txt.png)
If we run "sudo -l" we can see that webadmin can run a lua interpreter as sysadmin, so we can just run "os.execute('/bin/sh')" to get a shell.
![sudo -l webadmin](sudo_webadmin.png)
![Shell sysadmin](shell_sysadmin.png)
<h1>PrivEsc to Root</h1>
Running LinPeas we can see some interesting files that can be written in the /etc directory.
![LinPeas](linpeas.png)
These files are executed by root every time that someone login (in this case, using SSH). We don't have the password for the sysadmin user but we can write inside of .ssh/authorized_keys, which means that we can use a public and private key pair to authenticate without knowing the password.
![Generate key](ssh-keygen.png)
![Add key](add_key.png)
Now we have access to the machine using SSH, but we still need to get an interactive root shell. To do so I added a new entry inside /etc/sudoers that allowed sysadmin to run commands as root without suppling the password.
![00-header modified](modified-00.png)
![PrivEsc to root](privesc-root.png)
<br>
<h1>Conclusion</h1>
This machine is really easy even though is easy to miss the /etc/update-motd.d files if you don't know what they do. This is the second time that I solve this machine, just because I didn't take notes the first time and I wanted to have some writeups from HackTheBox.
