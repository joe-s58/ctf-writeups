# Linux Fundamentals Practice - Lab Writeup

Date: 06 Nov 2026 14:50

## The Scenario

A developer vanished, abandoning their Linux workstation. Word around the office is they hid something important before leaving. Your mission: SSH in and use your command-line skills to uncover the secret.

## Approach
 
Since the device IP is know, I started with a simple enumeration to find the developer's username and passwordd using http://target

## Findings

This was a simple cut-and-dry lab using basic linux commands once the username and password was located. The developer username and password were available using the IP address either using the Web Page or curl.  It turns out the web page was formatted in a way to ensure you could easily grab the username and password of the developer.

If using curl, things to search for are a key/value pair for anything with a username or password type label:

``` html
<div class="credentials-box">
      <h3>SSH Access</h3>
      <div class="credential-item">
        <span class="credential-label">Username</span>
        <span class="credential-value">XXXXXXXX</span>
      </div>
      <div class="credential-item">
        <span class="credential-label">Password</span>
        <span class="credential-value">XXXXXXXXXXXXXXXXXX</span>
      </div>
      <div class="credential-item">
        <span class="credential-label">Port</span>
        <span class="credential-value">22</span>
      </div>
    </div>
```

Once I had the login information, I was able to ssh into the system.

Traversing the user’s home directory revealed a sequence of hidden directories (files with  "." before the file name). You need to use the ls -a or ls -la to view hidden files. This ultimately provided access to the target file, which was also a hidden file.

## Timeline / IOCs

This section reconstructs the attack in the order it happened, then lists the
indicators of compromise (IOCs) a defender could use to detect the same activity.

## Timeline

| Step | Action | Evidence |
|------|--------|----------|
| 1 | Find the username and password | use the ip address and use the web or curl to read the user data in the html|
| 2 | Confirm ssh is available on the target | use nmap to confirm port, not really needed in this ctf as it was listed in the html |
| 3 | Login as the user via ssh | do recon in the users home directory structure looking for a flag file |
| 4 | View and copy the flag and submit it for the challenge | cat "filename" to view the file |


## Takeaway

As demonstrated in the laboratory exercise, not all data is immediately visible; however, hidden files are readily identifiable and accessible. Utilizing hidden files for storing sensitive information reflects poor data stewardship and inadequate privacy protection.