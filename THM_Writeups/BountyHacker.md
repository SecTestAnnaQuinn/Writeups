# Bounty Hacker

This is for the *easy* difficulty BountyHacker machine found at (https://tryhackme.com/room/cowboyhacker)
_____________
# Recon and Enumeration

First we will start with basic nmap analysis of the machine. `nmap -T4 Bounty`

This returns a few ports for us to look at: 21,22, and 80.  After viewing these, let's enumerate services on the ports to confirm they are the services we believe them to be. `nmap -T4 -sV Bounty -p 21,22,80`

![NMap](img/BountyHunter/NMap)

This does confirm the ports are ftp, ssh, and an Apache http server.

Let's go ahead and run a script to check for any apparent vulnerabilities in the systems by adding an -sC switch to our scan.

It reveals that the ftp port has anonymous allowed, and the http site does not have a title.

_________________
## Initial Foothold


Now we can check the ftp port for anything of use by remoting in with `ftp anonymous@Bounty`.  Here I find that I need to disable passive mode in ftp or else the traffic is insanely slow.  I do so by using the `passive off` command, then I `get` the two files we can see: task.txt and locks.txt.

![Cat](img/BountyHunter/Cat)

The files appear to be enumerating a user called lin and a list of potential passwords.  I would imagine we can use these together in a hydra query against the ssh port to try to crack the password and get in.  Let's go ahead and try that.

Our query should look something like this: `hydra -l lin -P /home/aquinn/locks.txt ssh://10.10.138.55 -T 64`

![InOnLin](img/BountyHunter/lin)

This gets us in on an ssh connection for Lin after we successfully find the password.  After which we can ls and find user.txt sitting right there.  
______________
## Privilege Escalation


Now it is time for everyone's favorite passtime!  Privesc!

Let's start by checking sudo privileges for fun.  `sudo -l`  

![BinTar](img/BountyHunter/Binnatar)

This reveals that the user may use /bin/tar as root.  This will likely be our in point.  Let's confirm with GTFOBins.  We should also check for extra SUID just in case.  `find / -user root -type f -perm -4000 2>/dev/null` should tell us what we need to know.

We don't see any exciting new vectors in the revealed commands, so let's go with tar for now.  In researching GTFObins, it is confirmed that we can run tar as sudo and keep the privileges with the command listed here `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`.  Let's put this in and get back to our roots.

![Root](img/BountyHunter/Root)

And we have done it!  In very short time too.  Now we just need to locate the file and call it a day.  We can do this with a quick `locate root.txt`.  Cat it out and put in your answers and you've got another box down.

![Nice](img/BountyHunter/crew)

