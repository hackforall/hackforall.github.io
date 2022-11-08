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

## CALCULATION OF EIP OFFSET

<br/>
<br/>
<br/>
> Curabitur blandit tempus porttitor. Nullam quis risus eget urna mollis ornare vel eu leo. Nullam id dolor id nibh ultricies vehicula ut id elit.

Etiam porta **sem malesuada magna** mollis euismod. Cras mattis consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla sed consectetur.

## Inline HTML elements

HTML defines a long list of available inline tags, a complete list of which can be found on the [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).

* **To bold text**, use `<strong>`.
* *To italicize text*, use `<em>`.
* Abbreviations, like <abbr title="HyperText Markup Langage">HTML</abbr> should use `<abbr>`, with an optional `title` attribute for the full phrase.
* Citations, like <cite>&mdash; Thiago Rossener</cite>, should use `<cite>`.
* <del>Deleted</del> text should use `<del>` and <ins>inserted</ins> text should use `<ins>`.
* Superscript <sup>text</sup> uses `<sup>` and subscript <sub>text</sub> uses `<sub>`.


Most of these elements are styled by browsers with few modifications on our part.

# Heading 1

## Heading 2

### Heading 3

#### Heading 4

Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor. Duis mollis, est non commodo luctus, nisi erat porttitor ligula, eget lacinia odio sem nec elit. Morbi leo risus, porta ac consectetur ac, vestibulum at eros.

--page-break--

## Code

Cum sociis natoque penatibus et magnis dis `code element` montes, nascetur ridiculus mus.

```php
// Example can be run directly in your JavaScript console
<?php

print($a+$b);
echo "hola";

?>

```

Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa.

## Lists

Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.

* Praesent commodo cursus magna, vel scelerisque nisl consectetur et.
* Donec id elit non mi porta gravida at eget metus.
* Nulla vitae elit libero, a pharetra augue.

Donec ullamcorper nulla non metus auctor fringilla. Nulla vitae elit libero, a pharetra augue.

1. Vestibulum id ligula porta felis euismod semper.
2. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus.
3. Maecenas sed diam eget risus varius blandit sit amet non magna.

Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.

Integer posuere erat a ante venenatis dapibus posuere velit aliquet. Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Nullam quis risus eget urna mollis ornare vel eu leo.

## Images

Quisque consequat sapien eget quam rhoncus, sit amet laoreet diam tempus. Aliquam aliquam metus erat, a pulvinar turpis suscipit at.

![placeholder](https://placehold.it/800x400 "Large example image") ![placeholder](https://placehold.it/400x200 "Medium example image") ![placeholder](https://placehold.it/200x200 "Small example image")

## Tables

Aenean lacinia bibendum nulla sed consectetur. Lorem ipsum dolor sit amet, consectetur adipiscing elit.

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Upvotes</th>
      <th>Downvotes</th>
    </tr>
  </thead>
  <tfoot>
    <tr>
      <td>Totals</td>
      <td>21</td>
      <td>23</td>
    </tr>
  </tfoot>
  <tbody>
    <tr>
      <td>Alice</td>
      <td>10</td>
      <td>11</td>
    </tr>
    <tr>
      <td>Bob</td>
      <td>4</td>
      <td>3</td>
    </tr>
    <tr>
      <td>Charlie</td>
      <td>7</td>
      <td>9</td>
    </tr>
  </tbody>
</table>

Nullam id dolor id nibh ultricies vehicula ut id elit. Sed posuere consectetur est at lobortis. Nullam quis risus eget urna mollis ornare vel eu leo.
