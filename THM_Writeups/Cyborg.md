# Cyborg

## Enumeration

1. As always, we start with an nmap with some intensive switches to view port and service info

```shell
nmap -sV -sC -Pn
```
This reveals the following information
```shell
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-21 20:07 MDT
Nmap scan report for *ip**
Host is up (0.18s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 db:b2:70:f3:07:ac:32:00:3f:81:b8:d0:3a:89:f3:65 (RSA)
|   256 68:e6:85:2f:69:65:5b:e7:c6:31:2c:8e:41:67:d7:ba (ECDSA)
|_  256 56:2c:79:92:ca:23:c3:91:49:35:fa:dd:69:7c:ca:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2.  - We see that ssh and http are both open, with Apache 2.4.18 running.  
    - Checking the website leads to an Apache2 default page, let's see if they started working on other pages first.  
    - Gobuster time!

## Discovery
```shell
/admin                (Status: 301) [Size: 312] [--> http://10.10.157.38/admin/]
/etc                  (Status: 301) [Size: 310] [--> http://10.10.157.38/etc/]  
```

1. It appears there is an admin page we can get to, let's check it out.

2. Admin takes us to what appears to be a blog page, while etc takes us to an index with a folder called squid/

3. Checking out squid we find a password.txt with a nicely provided credential set.

`music_archive:*password provided here*`

We also find a file for configuration of squid 

______________________________


4. Let's go back to the admin page and poke around

There is a dropdown in the top right with many options 
```shell
    Home
    Albums
    Admins
    Archive
        Listen
        Download
```

Under Archive, clicking on Listen does nothing, but clicking on Download gives us archive.tar.  Let's store that for later.

Admin takes us to an Admin Shoutbox which gives us some chat logs
```shell

    Admin Shoutbox

            
                ############################################
                ############################################
                [Yesterday at 4.32pm from Josh]
                Are we all going to watch the football game at the weekend??
                ############################################
                ############################################
                [Yesterday at 4.33pm from Adam]
                Yeah Yeah mate absolutely hope they win!
                ############################################
                ############################################
                [Yesterday at 4.35pm from Josh]
                See you there then mate!
                ############################################
                ############################################
                [Today at 5.45am from Alex]
                Ok sorry guys i think i messed something up, uhh i was playing around with the squid proxy i mentioned earlier.
                I decided to give up like i always do ahahaha sorry about that.
                I heard these proxy things are supposed to make your website secure but i barely know how to use it so im probably making it more insecure in the process.
                Might pass it over to the IT guys but in the meantime all the config files are laying about.
                And since i dont know how it works im not sure how to delete them hope they dont contain any confidential information lol.
                other than that im pretty sure my backup "music_archive" is safe just to confirm.
                ############################################
                ############################################
```

Turns out we were on the right track!  It appears that there may also be a backup of the music_archive somewhere as well we may be able to get to.


5. -  Let's look at the .tar, moving to our folder we can go ahead and untar it with tar -xvf
    - this gives us many different files, let's look around.
    - Within the folders we find a Readme denoting that the archive is a BORG archive.  We also see that there are many config files, and index appears to be encrypted.
    - Digging down into the data provided by this, cat'ing 1 and 3 gives nothing but 4 looks as though it may be a binary?  let's examine it in xxd.

There's a lot of data, but none useful.  Guess we'll have to try figuring out how to deborg the stuff with the username and password we found earlier.

___________

## Vulnerability Assessment

1. - We review the Borg documentation and find that you can restore a backup using borg, so let's try installing that.

    - Now we can go to the file for dev and check final_archive, but it needs a passphrase for the key, luckily we have that!

## Exploitation

1. Let's crack the hash real quick with John.  This leads us to the password *redacted*.  When we try using that to check final_archive using borg, we can see that it is indeed confirmed to be a borg archive.  

2.  Let's create a new folder and mount the repository, then explore.

We now have access to alex's backup of his drives, time to poke around!

3. In Desktop we find a nice note:
```shell
    shoutout to all the people who have gotten to this stage whoop whoop!"
```
:)

In documents we find something more juicy though:
   ```shell
    Wow Im awful at remembering Passwords so Ive taken my Friends advice and noting them down!

    alex:S3cretP@s3
```

Nothing else is in the backup (WHY WOULD YOU BACK THIS UP WITH ONLY SENSITIVE INFO AND PUT IT ONLINE)

4. Let's immediately try to ssh in using alex and the pasword

It works and we can see the juicy user.txt right in front of our noses.
    `flag{*flag*}`

Now we can explore around a bit more.

5.  - Many folders are empty, but Music has a lot of mp3 files suspiciously marked image, let's nab them all - for science.

    - They all have root permissions and no data that we can see, let's carry on :(

Nothing of note, time for us to explore and try to find privesc.

6. `Sudo -l` returns that alex can use sudo on `/etc/mp3backups/backup.sh`.  since this is a shell, it is pretty easy to use it to get root, let's use that as an escalation point.  We echo /bin/bash to the shell after chmodding it to have full permissions and we can then become root

## Final Flag

Now we head to root, and find exactly what we need: root.txt
    flag{*rootflag*}


