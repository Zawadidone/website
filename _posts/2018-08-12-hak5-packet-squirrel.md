---
name: Hak5 Packet Squirrel
date: 2018-08-12
header: 
  teaser: /assets/img/packetsquirrel.jpg
---

A week ago, I ordered some [tools](https://twitter.com/ZawadiDone/status/1024528907729559552) from
[Hak5](https://hakshop.com/). The Lan turtle and the Wifi Pineapple. I always watch the video’s from
the [Hak5 Youtube channel](https://www.youtube.com/channel/UC3s0BtrBJpwNDaflRSoiieQ) and saw them playing with the tools and, why not give it a chance. The Packet Squirrel includes three out of the box payloads for logging packets to USB drives, spoofing DNS and tunnelling out through a VPN.

[![Hak5 Packet Squirrel]({{site.url}}/assets/img/packet_squirrel_diagram2.png){:height="300px" width="600px"}]()

### Logging packets to USB drives
Turn the switch of the Packet Squirrel to the first position. Plug the network cables and micro USB cable in the Packet Squirell is shown on the image above. The tcpdump payload will write a pcap file to a connected USB disk until the disk is full. A full disk will be indicated by a solid green LED. After unplugging the cables you can remove the USB flash drive and inspect the stored pcap file with a network protocol analyzer such as Wireshark.

I used this on an assignment for work. We were doing an internal pentest but the network used MAC
address authentication. Please don’t use that because of [MAC Spoofing](https://linuxconfig.org/how-to-change-mac-address-using-macchanger-on-kali-linux). We intercepted the packets with the Packet Squirrel, opened the pcap file with Wireshark and got the MAC-address of the printer. Spoofed our own MAC address and we got access to the internal network.

### DNS spoofing google.com
To configure the DNS Spoof payload with custom mapping, just power on the Packet Squirrel in Arming Mode (switch to the far-right position) and edit the file '/root/payloads/switch2/spoofhost'. Replace # with the domain (google.com) you wish to spoof, and the IP address with the spoofed destination.

With the file (spoofhost) configured and saved, power off the Packet Squirrel and flip the switch to position 2. Now place the Packet Squirrel inline between a target and the network. When it powers on the DNS spoof payload will run, indicated by a single blinking yellow LED. Here a [demo](https://twitter.com/ZawadiDone/status/1026162984714944513) of spoofing google.com on my laptop. For more info about the Packet Squirrel here is the [documentation](https://www.hak5.org/gear/packet-squirrel/docs).
