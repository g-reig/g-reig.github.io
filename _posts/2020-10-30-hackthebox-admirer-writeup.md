---
layout: post
title: "HackTheBox: Admirer WriteUp"
categories: [write-ups]
permalink: /write-ups/htb_admirer_writeup/
---
Admirer is an easy rated machine hosted in HackTheBox. As always, the goal is to obtain root and read the flag in /root/root.txt.<br>
It's been a long time since I've posted something, that is because I am really busy with college right now so I don't have time to solve CTFs and do the write-ups.
<h1>Port Scan</h1>
![Port Scan](portscan.png)
We can see that there are 3 ports open with ftp, ssh and http.
<h1>HTTP Enumeration</h1>
Looking at the default scripts output from Nmap we can see that there is a robots.txt file with a disallowed entry.
![Robots](robots.png)
The entry is a directory without directory listing enabled, so I used Gobuster to enumerate the files.
![Gobuster Admin Dir](gobusterAdminDir.png)
There are two files, one with some names and emails and the other one with some credentials.
![Contacts](contacts.png)
![Credentials](credentials.png)
<h1>FTP Authenticated Login</h1>
FTP doesn't have anonymous access enabled but the credentials obtained in the credentials.txt file allow us to login as ftpuser to the server. There are only two files inside the FTP server (a database dump and a tar.gz file).
![FTP File List](ftpFiles.png)
The database dump doesn't have anything really interesting, but it tells us the database that is being used and some info about the server.
![SQLDump](sqldump.png)
The html.tar.gz seems to be an old backup of the web on port 80.
![HTML list](htmlList.png)
The w4ld0s_s3cr3t_d1r seems to be the same as admin-dir and the utility-scripts directory has some administration related php scripts.
![Utility Scripts ls](utilityScriptsLs.png)
Inside of the db_admin.php file we can read the following:
![DB Admin Code](db_adminCode.png)
<h1>Finding Database Admin</h1>
The utilty-scripts directory exists on the web server but it doesn't have the db_admin.php file, which means that they've found an open source alternative. I used both Gobuster and Google to find the necessary file.
![Google DBADMIN](google.png)
The first search result on Google shows "Adminer", which is also the name of the database (it is written in the SQLDump).
Gobuster found a file called adminer.php inside of the utility-scripts directory.
![Gobuster utility-scripts](gobusterUtScr.png) 
<h1>Adminer Enumeration</h1>
The credentials of the database are inside the index.php file in the backup, but they don't work with the Adminer page.
![Backup credentials](indexBakCred.png)
![Login failed](loginFail.png)
As we can see in the login failed image the installed version of Adminer is 4.6.2, which is [vulnerable and leads to file disclosure](https://www.foregenix.com/blog/serious-vulnerability-discovered-in-adminer-tool).
<h1>Exploiting Adminer</h1>
To exploit the vulnerability we need to have a mysql server exposed to the network with users that have remote access and permissions to write in a table.<br>
I changed the bind address inside the /etc/mysql/mariadb.conf.d/50-server.cnf file.
![Bind Address](adminerexp1.png)
![Starting the server](adminerexp2.png)
![Create user and GRANT](adminerexp3.png)
![Create the table](adminerexp4.png)
Once all the preparations are done we only need to connect to our server using the Adminer of the machine.
![Adminer Login](adminerLogin1.png)
![Adminer Login Success](adminerLogin2.png)
Then we can execute this query with the index.php file to dump the current database credentials into our SQL server:
{% highlight SQL %}LOAD DATA local INFILE "nameOfFile" INTO TABLE file;{% endhighlight %}
![Query Executed](query1.png)
![Credentials](query2.png)
<h1>SSH as Waldo</h1>
The credentials found using the file disclosure can be used to access the machine as the waldo user.
![SSH](sshWaldo.png)
Running the "sudo -l" command we can see that waldo can execute the "/opt/scripts/admin_tasks.sh" as root while preserving his environment variables.
![Sudo -l](sudo-l.png)
This code calls a python script that backups the web files:
{% highlight bash %}backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}
{% endhighlight %}
The python script imports a function from the shutil module and executes it with three arguments:
{% highlight python %}#!/usr/bin/python3

from shutil import make_archive

src = '/var/www/html/'

# old ftp directory, not used anymore
#dst = '/srv/ftp/html'

dst = '/var/backups/html'

make_archive(dst, 'gztar', src)
{% endhighlight %}

<h1>PrivEsc to Root</h1>
Reading the python documentation I found that python has a variable that specifies the search path for the modules, which is initialized using the PYTHONPATH environment variable. If we change the PYTHONPATH we can inject a malicious module and run it as root.
![PYTHONPATH info](PYTHONPATHinfo.png)
I wrote this module, which grants waldo sudo privileges to run commands as every user (I tried using a reverse shell, but it didn't work because the process was getting sent to background).
{% highlight python %}def make_archive(arg1,arg2,arg3):
    f = open("/etc/sudoers","a")
    f.write("waldo    ALL=(ALL:ALL) ALL")
    f.close()
{% endhighlight %}
![PrivESC](privesc.png)

<h1>Conclusion</h1>
This machine focuses on enumerating the web server, which was an unknown topic for me the first time I attempted it. If you are familiar with this then the machine is pretty straightforward until the privilege escalation part, where I had to spend some time because I didn't know anything about the python module importing mechanism.
