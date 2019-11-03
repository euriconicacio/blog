---

layout: post

title: Day 7 of 100 - Wifi Router MAC Address in 2 methods

---

Hey guys, differently from what I said before, today I will make a pause in the Google Hacking theme just to answer online a question made in person by a friend when I talked him about what we discussed here on [Day 5 of 100](https://euriconicacio.github.io/blog/Day-5-Wordlists-crunch-parte-1/), concerning the correlation of the Wifi router MAC address/BSSID and its default password. So, this friend of mine asked me: **_"Is there any way to discover a Wifi router MAC address WITHOUT BEING CONNECTED to it?"_**. 

First and quick answer: **obviously yes**, with a little remark: _we are talking about BSSID, and CONCEPTUALLY it is not exactly equal to MAC address_ (for a further explanation, check [this](https://community.arubanetworks.com/t5/Controller-Based-WLANs/How-is-the-BSSID-derived-from-the-Access-Point-ethernet-MAC/ta-p/176290)). However, in order to make it simpler, I shall vary its name through this post - using both MAC address and BSSID - just because I'm stubborn ~~as hell~~. 

I will cite here two (of lots of) methods for discovering such address - one of them, well-known in hacker culture world, and the other with a *self open source solution in Python*. So, lets see these two solutions.

---

### Method 1 - Classic _"h4x0r"_ airmon-ng + aireplay-ng borgin method

I called this a _classic 'h4x0r'_ method just because it might seem simple to any _script kiddie_ perform by following online "how-to-_honk_-my-neighbour-wifi" tutorial (pun intended). It only demands two or three basic steps, depending on your current Linux distro ~~(always use Linux)~~ configuration:

1. Install _Aircrack-ng_ suite with `sudo apt install aircrack-ng` _[skip this step if you already have it installed or read [this masterpiece](https://www.aircrack-ng.org/doku.php?id=newbie_guide) if you don't know what aircrack-ng is]_
   
2. Enable monitor mode on your wireless interface with **airmon-ng**:
   * _Skipping some basic steps since this is not the main theme for this post_, this procedure is easily performed with `sudo airmon-ng start <wireless_interface_name>`;
   * To check your wireless interface name, just type `sudo airmon-ng` and check it out; regular names are _wlan0_, _wlp2s0_ and others;
   * Let's assume your wireless interface name is _wlan0_, for instance. After starting monitor mode, its name might change to _wlan0mon_.
3. Start package capturing of raw 802.11 frames on the wireless interface under monitor mode with **airodump-ng**:
   * _Once again, skipping some basic steps because of reasons_, just type `sudo airodump-ng start wlan0mon` (since we assumed _wlan0_ was your wireless interface name and _wlan0mon_ its monitoring mode name), press enter and hold;
  
   <img src="https://tenor.com/view/braveheart-hold-on-gif-13268605">

> _"HOLD... HOLD... HOOOOLD... HOOOOOOOOOOLD... **NOOOOOOWWWW!**"_ - William Wallace, Braveheart (1995)

   * After some time, you can stop airodump execution and there you go: a list of BSSID/ESSID of reachable wireless networks. See an example (taken from the internet) for an output of airodump-ng:
  
<img src="https://img.wonderhowto.com/img/25/91/63509567831583/0/hack-wi-fi-getting-started-with-aircrack-ng-suite-wi-fi-hacking-tools.w1456.jpg">
   
   * **FTR #1**: that is _one of the simplest things_ someone can do with airodump-ng. For instance, if you have a GPS receiver connected to the computer, airodump-ng is capable of logging the coordinates of the found access point. Learn more [here](https://www.aircrack-ng.org/doku.php?id=airodump-ng).

4. After this procedure, execute `sudo service network-manager restart` to return your wireless interface from monitor to regular mode.

---

### Method 2 - Self open source solution in Python

I have spent some time on this subject and developed a simple Python3 routine that uses [Scapy](https://scapy.net/) 802.11 packets handling features to capture BSSID/ESSID of reachable wireless AP signals. In order to run it, install scapy by running:

```bash
sudo pip3 install scapy
```

After that, feel free to run the code ahead under root privilege:

```python
from scapy.layers.dot11 import Dot11
from scapy.sendrecv import sniff
def sniffer(pck):
    if pck.haslayer(Dot11):
        if pck.type == 0 and pck.subtype == 8:
                print("AP BSSID: %s - SSID: %s" %(pck.addr2, pck.info))
sniff(iface="wlan0", prn=sniffer, store=0)
```

That code shall print a list of AP BSSID/ESSID similar to those in airodump-ng. In case of issues, check dependencies, privileges and other obstacles; it works really fine for me. =P

---

And, in the end, _**why is that important at all**_? For now, let's just consider a Brazilian particular case for default wireless network naming in main internet providers: GVT/Vivo and NET/Claro use part of the router BSSID/MAC as its default wireless network names! By knowing this fact and discovering the upmentioned address, and also by hoping someone who did not change the wifi default name might not have changed its default password  one might easily **crack the wifi password** simply by typing **the last 8 characters of the router BSSID**. 

For instance, let's say that after using airodump-ng, I get as an reachable BSSID/ESSID a wireles network named **"NET_2GB8373B"** and addressed as **"98:1E:19:B8:37:3B"**. For _gods_ sake, the default password for that network totally is _**19B8373B**_ (**FTR #2**: I would try that even for the network on the example image above with ESSID _"HOME-**2988**"_ and BSSID _B8:9B:C9:59:**29:88**_. See the pattern?!)

So, this is a tested and approved method for "guessing" passwords of wireless networks with default ESSIDs through two different methods. Hope you all enjoy this post and next time (I will work to keep schedule and post until Monday..), I promise I will talk about Google Hacking. See you then!