# EasyCTF

This is for the *easy* difficulty EasyCTF machine found at (https://tryhackme.com/room/easyctf)
_____________
# Recon and Enumeration

First we will run a basic scan with nmap to enumerate any suspicious services.

`nmap -T4 EasyCTF` reveals that we are working with ports 21, 80, and 2222.  As we review this, lets do another scan to tell us about the services on the ports.

`nmap -T4 -sV -p 21,80,2222 EasyCTF`

![NMap](img/EasyCTF/NMap)

It appears as though 2222 is an ssh port.  First we'll go ahead and start one more scan with the -sC switch as well, to check for any vulnerabilities.

This extra switch tells us that the Anonymous FTP login is allowed, which is great for us.  It also mentions that robots.txt has 2 disallowed entries, so we may want to check that out as well.

![Robots](img/EasyCTF/Robots)

This enumerates a user listed as mike, which could be handy information for us in a moment.  We will likely be able to run hydra with the username to brute force ftp or ssh.  In addition, it tells us of a /openemr-5_0_1_3 page we should visit.  In addition let's start up gobuster to see if we find anything else.

`gobuster dir -w /usr/share/wordlists/dirb/common.txt -t 30 -u 10.10.157.82`

![Gobuster](img/EasyCTF/Gobuster)

The openemr directory showed us nothing, but gobuster does show us a /simple directory we can connect to.  As we do that, let's get hydra running against ssh for user mike and see what comes up.  Remember the -s switch with a 2222 definition as the ssh port we found was on 2222 and not the default 22.

`hydra -l mike -P /usr/share/wordlists/rockyou.txt ssh://10.10.157.82 -T 200 -s 2222`
_____________
# Vulnerability Analysis

As we explore the http site, visiting the /simple directory sends us to a cms site for cms made simple.  

The site on its own doesn't tell us much, but it does have the following text:

```text
To get to the Administration Console you have to login as the administrator 
(with the username/password you mentioned during the installation process) on your site
at http://yourwebsite.com/cmsmspath/admin. If this is your site click here to login.
```
An admin login page does sound pretty juicy, let's check that out as well.

![cms](img/EasyCTF/cms)

Ahhhh, yet another chance to brute force the login.  Let's do some recon first.  In using the inspect element and reviewing the page request, it appears as though the request is made with http post.  So we will put that into our hydra command.  -l mike is setting the user to the mike credential we found earlier.  -P uses a wordlist instead of a single password and we will supply that the rockyou.txt file.  The IP is obviously the ip of the machine.  

Now comes a trickier part.  If you don't know how to find the request body, that's okay, neither did I.  I did find help for this on https://infinitelogins.com/2020/02/22/how-to-brute-force-websites-using-hydra/.  You can either use their instructions for it, or the steps I have found.  This would be under network there will be a request tab.  Click the raw slider into the on position and you will see the payload.

Now we format the command as follows:

`sudo hydra <Username/List> <Password/List> <IP> <Method> "<Path>:<RequestBody>:<IncorrectVerbiage>"`

Given everything we found on the site, that would then be something like:

`sudo hydra -l mike -P /usr/share/wordlists/rockyou.txt 10.10.157.82 http-post-form "/simple/admin/login.php:username=mike&password=^PASS^&loginsubmit=Submit:User name or password"`

Let's try that!  Hopefully this works for us, as the ssh attempts have not been going great thus far.

As we run this, let's check on the ftp finally.  We'll use the command `ftp anonymous@easyctf` as anonymous was enabled for the ftp port.  In the system we can see only one directory: pub.  We cd down to it and find ForMitch.txt.

![Mitch](img/EasyCTF/ForMitch)

This enumerates a user called Mitch who apparently has a very weak password.  Let's run the same ssh hydra query on his.  Unfortunately no results are turning up for the other hydras so far.

`hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://10.10.157.82 -T 200 -s 2222`

That was a quick hit!  We now have mitch's password.  Let's try to get into ssh using it.

______________
## Privilege Escalation

Yep!  We are in and can get the user flag here.  We also see another user named sunbath, but let's try to get some privesc going on.  A search for suid's gives us the jackpot and we can now see that the user has permissions to run vim sudo passwordless as root.  A check on GTFOBins confirms we can run the following command to get out - make sure you use the address listed under the permissions exactly, just running vim may not work.

`sudo /usr/bin/vim -c ':!/bin/sh'`

![Root](img/EasyCTF/Privesc)

We are now root!  Let's go ahead and locate root.txt with that exact command, and we can cat it out from the root directory to get our final key.  


