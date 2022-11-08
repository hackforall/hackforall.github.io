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

Welcome to HackForAll Blog.

Today, I'm going to solve the BRAINPAN 1 CTF published in the plataform VULNHUB.

The purpose of this machine is to learn how to exploit the stack doing a buffer overflow in a vulnerable
binary file.

So, let's start with it.

<br/>

# Requirements

* VMWARE or similar.
* A Offensive Linux Machine (Kali Linux or Parrot)
* Windows Machine
* Python2.7
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
