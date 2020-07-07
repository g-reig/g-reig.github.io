---
layout: post
title:  "PwnLab: Init Write-Up"
categories: [write-ups]
permalink: /write-ups/pwnlab_init_writeup/
---
PwnLab is really easy boot to root VM hosted on Vulnhub. This is the last easy machine in the [abatchys list](https://www.abatchy.com/2017/02/oscp-like-vulnhub-vms)
(I don't have the write-ups for the first four levels of Kioptrix, but I don't feel like doing them again just for the write-up). The goal is to get root in the machine and read the flag.
<h1>Network Scan</h1>
As always I used Nmap to do a ping scan on the entire subnet and get the IP of the vulnerable box.
![Network Scan](networkscan.png)
In this case, the IP was 192.168.56.129
<br><br>
<h1>Port Scan</h1>
Doing the port scan (using Nmap again) we can see that there aren't a lot open ports. The most interesting ones are 80 and 3306 (HTTP and MySQL). I couldn't find what was in port 54852, but it wasn't necessary to get the foothold.
![Port Scan](portscan.png)
<br>
<h1>Web Enumeration</h1>
From the Nmap default scripts output we can see that the port 80 has an Apache 2.4.10 webserver, which doesn't appear to be vulnerable to anything.<br>
Running gobuster we can see some php pages, being the config.php the most interesting one because usually configuration files contain credentials.
![Gobuster](gobust.png)
In the webserver we can see the following pages:
![rootDir](rootDir.png)
![Login page](loginPage.png)
The login page isn't vulnerable to SQL Injection, but the root page is vulnerable to Local File Inclusion (LFI). Using [a trick that I found on the Internet](https://highon.coffee/blog/lfi-cheat-sheet/) I was able to read the source code for all the php files.
<h3>Contents of config.php:</h3>
![LFI Config 1](LFIconfig1.png)
![LFI Config 2](LFIconfig2.png)
Here we can read the credentials of the root user in the MySQL database.
<h3>Contents of index.php:</h3>
{% highlight php %}<?php
//Multilingual. Not implemented yet.
//setcookie("lang","en.lang.php");
if (isset($_COOKIE['lang']))
{
        include("lang/".$_COOKIE['lang']);
}
// Not implemented yet.
?>
<html>
<head>
<title>PwnLab Intranet Image Hosting</title>
</head>
<body>
<center>
<img src="images/pwnlab.png"><br />
[ <a href="/">Home</a> ] [ <a href="?page=login">Login</a> ] [ <a href="?page=upload">Upload</a> ]
<hr/><br/>
<?php
        if (isset($_GET['page']))
        {
                include($_GET['page'].".php");
        }
        else
        {
                echo "Use this server to upload and share image files inside the intranet";
        }
?>
</center>
</body>
</html>
{% endhighlight %}
Here we can see two sources of LFI, the first one being the lang cookie and the second one being the page GET argument. The one in the cookie allows us to include files with an extension different from .php.
<h3>Contents of login.php:</h3>
{% highlight php %}<?php
session_start();
require("config.php");
$mysqli = new mysqli($server, $username, $password, $database);

if (isset($_POST['user']) and isset($_POST['pass']))
{
        $luser = $_POST['user'];
        $lpass = base64_encode($_POST['pass']);

        $stmt = $mysqli->prepare("SELECT * FROM users WHERE user=? AND pass=?");
        $stmt->bind_param('ss', $luser, $lpass);

        $stmt->execute();
        $stmt->store_Result();

        if ($stmt->num_rows == 1)
        {
                $_SESSION['user'] = $luser;
                header('Location: ?page=upload');
        }
        else
        {
                echo "Login failed.";
        }
}
else
{
        ?>
        <form action="" method="POST">
        <label>Username: </label><input id="user" type="test" name="user"><br />
        <label>Password: </label><input id="pass" type="password" name="pass"><br />
        <input type="submit" name="submit" value="Login">
        </form>
        <?php
}

{% endhighlight %}
Reading the source code we can confirm that it isn't vulnerable to SQL Injection and that the passwords are base 64 encoded inside of the database.
<br><br>
<h1>MySQL Enumeration</h1>
We can access the MySQL server using the credentials inside of the config.php file. 
![Accessing mysql](mysql1.png)
There is only one interesting database with a single table which has the usernames and the passwords for the website login page (in base 64).
![MySQL 2](mysql2.png)
<br>
<h1>Remote Code Execution (Low Privilege Shell)</h1>
We get access to the upload page once we log in. From here we can upload a php reverse shell disguised as a GIF image to bypass the file type restrictions (I explained this method in the FristiLeaks 1.3 Write-Up).<br>
Once we upload the shell, we can get its path inside of the source code.
![Uploading shell](shellupload.png)
![Shell path](shellpath.png)
Using the LFI in the lang cookie we can execute the reverse shell.
![LFI2RCE](lfi2rce.png)
![lowshell](lowshell.png)
<br>
<h1>PrivEsc to Mike</h1>
Both Kent and Kane use the same credentials as they did in the webpage, so we can use the "su" command to change our user. Kent doesn't have anything useful, but Kane has a SetUID binary in the home directory.
![msgmike](msgmike.png)
If we decompile the binary, we can see that is executing "cat /home/mike/msg.txt".
![msgmike decompiled](msgmikedecomp.png)
As the binary is executing "cat" using a relative path, we can create a script called "cat" in a directory which is in our PATH environment variable to execute our code instead of the intended one.
![privesc to mike](mikeprivesc.png)
<br>
<h1>PrivEsc to Root</h1>
In the Mikes home directory we can see another SetUID binary, but this one is owned by root.
![msg2root](msg2root.png)
Decompiling the binary we can see that it uses asprintf("/bin/echo %s >> root.txt",...), so we can inject our code in there.
![msg2root decompiled](msg2rootdecomp.png)
In this case, I decided to compile a C program that executes "/bin/bash" as root (I know that there are easier ways, but I wanted to try this).<br>
{% highlight C %}#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char** argv) {
        if (setreuid(0,0) < 0) {
                perror("Setuid\n");
                exit(1);
        }
        if (system("/bin/bash")) < 0) {
                perror("Error executing command\n");
                exit(1);
        }
}
{% endhighlight %}
![Privesc2root](privesc2root.png)
<br>
<h1>Conclusion</h1>
This is a really easy box and I didn't have any trouble solving it, mainly because I knew all the tricks that were needed except for the LFI one.<br>
As I said before, this is the last easy box in the list, so I don't know if I'm going to keep solving the next boxes or if I'm going to do more specific challenges to learn more before trying them (probably binary exploitation or Windows).
  
