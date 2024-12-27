# OpenWRT Journey

## Firmware installation
One thing that complicated updating the firmware from the factory version is that the openwrt One 
router is very new. As a result there wasn't a build for the current stable release (23.05). 
I mistakenly used a development snapshot which doesn't have the GUI available.

I was still able to SSH to the device and use `sysupgrade` to install a release candidate version. The 
[firmware-selector.openwrt.org](https://firmware-selector.openwrt.org/?version=24.10.0-rc4&target=mediatek%2Ffilogic&id=openwrt_one) 
site is great. It's indexed by version so you have to flip through them to find one that has `OpenWrt One`.

## Getting access working prior to wifi being enabled
I didn't want to enable wireless until I had good defaults. The problem with this is that at least 
when I was setting up the save button on the wireless tab didn't work. This meant I had to have 
the device hardwired into my PC and also needed wifi work so I could still access the internet.
The problem was that my PC prioritizes internet from the LAN/Ethernet cable first. So it was really annoying
to look for anything since the page would take an extra 5-10 seconds to load. I figured out how to
change the prioritization of interfaces in windows:

* First you need to know the interface priority. This can be found in windows Powershell `Get-NetIPInterface`
* For me Ethernet was 25 and wifi was 35. What I wanted was for ethernet to be after wifi.
  That way when I got to a webpage it would try the wifi interface first. So I am going to change
  the "InterfaceMetric" to be higher for Ethernet.
* `WIN+R` > `ncpa.cpl` 
* Right click in the ethernet interface > Properties
* Select `Internet Protocol Version 4 (TCP/IPv4)` > Properties > Advanced
* Uncheck the "Automatic metric" and put in the interface metric that makes sense for your system.
  In my case 40.
* Then repeat for `Internet Protocol Version 6 (TCP/IPv6)`

One you do this now you have a new problem. If you go to connect to the OpenWRT router using 
the IP address 192.168.1.1 which was working before it now will use your wifi and send you to 
the existing wifi router page. So what we need to do explicitly tell the browser and ssh to 
connect via the ethernet interface even though the wifi interface is a higher connection priority.

Turns out each interface on your windows computer has an IP address I guess? I don't really understand 
how that works. In a windows powershell session I ran `ipconfig`. From there I got this output (I
cut some bits out).

```
> ipconfig

Windows IP Configuration

Wireless LAN adapter Wi-Fi:

   Connection-specific DNS Suffix  . : tds.net
   Link-local IPv6 Address . . . . . : fe80::6475:9e94:fc1d:4edd%20
   IPv4 Address. . . . . . . . . . . : 192.168.1.193
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : lan
   IPv6 Address. . . . . . . . . . . : fdac:5f89:c6d3::bc3
   IPv6 Address. . . . . . . . . . . : fdac:5f89:c6d3:0:2f0e:c420:f86:8c10
   Temporary IPv6 Address. . . . . . : fdac:5f89:c6d3:0:59c2:d94a:ea5b:f38
   Link-local IPv6 Address . . . . . : fe80::4c5f:8e35:3b0f:976a%9
   IPv4 Address. . . . . . . . . . . : 192.168.1.127
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

The ethernet device reports it's IPv4 as 192.168.1.127. I now want to tell my windows computer to connect
to the interface on 192.168.1.127 but once there connect to 192.168.1.1 which is the default 
openwrt IP address. Googling for SSH commands to do this wasn't super helpful. The bit that finally got 
me there was looking for the word bind address. I asked copilot 
_using ssh connect to a remote server and allow for forwarding traffic to it using a bind addresss_. 
This returned

> To connect to a remote server via SSH and allow for forwarding traffic using a bind address, you can use the `-L` option in your SSH command. This option specifies the local port to bind to, the address to bind on, and the remote port to forward traffic to. Here’s a basic example:
> 
> ```sh
> ssh -L [local_bind_address]:[local_port]:[remote_address]:[remote_port] user@remote_host
> ```
> 
> Here’s a breakdown of the parameters:
> - `[local_bind_address]`: The local address to bind to. It’s often `localhost`.
> - `[local_port]`: The local port to bind to (e.g., `8080`).
> - `[remote_address]`: The address of the destination machine on the remote network.
> - `[remote_port]`: The port on the destination machine you wish to access.
> - `user`: Your username on the remote host.
> - `remote_host`: The remote SSH server’s address.
> 
> Example:
> ```sh
> ssh -L localhost:8080:192.168.1.100:80 user@remote_host
> ```
> 
> In this example, traffic sent to `localhost:8080` on your local machine will be forwarded to `192.168.1.100:80` on the remote server.

First to just ssh to router I used the following. I use the bind address of the interface but then
go to 192.168.1.1

```
ssh -b 192.168.1.127 root@192.168.1.1
```

To get the webpage working I used: 

```
ssh -L localhost:6789:192.168.1.1:443 -b 192.168.1.127 root@192.168.1.1
```

Now I can get to OpenWRT LUCI web page via https://localhost:6789/.

## Hardening
I stared with this page. https://austindw.com/harden-openwrt/
* I did restrict SSH access to lan only. This also restricts access from the guest network.
* I'm going to leave password auth in place. Idk if it's restricted it's nice to be able to save 
  off the password and use it to connect in case I lose my private key. I don't like to save private 
  keys in password managers.
* I did force https and restrict Luci to only access over lan and HTTPS
* I did DROP external connections rather than REJECT

## Getting wireless working

### Guest network 
I want a guest network that I'll actually just use for IoT devices. I want something where the IoT devices
can't talk to each and can't talk to more sensitive devices on my network like my computer, phones, etc.
The page on guest networks on openwrt is quite good. 

I also will use the security WPA2 access and allow for a bunch of the backwards compatible access. 

https://openwrt.org/docs/guide-user/network/wifi/guestwifi/guest-wlan

