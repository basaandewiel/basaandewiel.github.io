---
layout: post
title: Wireguard VPN on openWRT router behind ISP router
---
Used sources
* https://forum.openwrt.org/t/wireguard-server-on-openwrt-router-behind-isp-router-firewall-config/189027/20
* https://openwrt.org/docs/guide-user/services/vpn/wireguard/basics


# Goal and introduction
If I am away form home, for instance on holiday, I want to have access to my LAN, including all equipment attached to the LAN.
I was using Openvpn on openWRT for that. That was working.
But after upgrading openwrt I lost the configuration of openvpn, and I knew it was a big hassle to get it working again.

I read somewhere that *wireguard* should be better and simpler. So I decided to give wireguard a try.

The configuraion appeared to be not that simple, at least not if you are not a network expert.

I tried for several days to get it working myself, but get stuck. So finally I asked for help in the openwrt forum. And there are a lot of experts willing to help, and I got in working within one day.
This post describes what I have done. I hope it will be helpfull for others.


# Situation
Equipment used
* OpenWrt 23.05.0 on Linksys WRT3200ACM

Configuration
* ISP router
  * 192.168.2.254

* openwrt router
  * 192.168.2.253 (WAN interface)
  * 192.168.1.1 (internally)


# How wireguard works
WireGuard securely encapsulates IP packets **over UDP**. You add a **WireGuard interface**, configure it with **your private key and your peers' public keys**, and then you send packets across it. All issues of key distribution and pushed configurations are out of scope of WireGuard; these are issues much better left for other layers, lest we end up with the bloat of IKE or OpenVPN. In contrast, it more mimics the model of SSH and Mosh; *both parties have each other's public keys, and then they're simply able to begin exchanging packets through the interface.**

Each network interface has a private key and a list of peers. 
WireGuard associates **tunnel IP addresses with public keys and remote endpoints**. When the interface sends a packet to a peer, it does the following:
* This packet is meant for 192.168.30.8. Which peer is that? Let me look... Okay, it's for peer ABCDEFGH. (Or if it's not for any configured peer, drop the packet.)
* Encrypt entire IP packet using peer ABCDEFGH's public key.
* What is the remote endpoint of peer ABCDEFGH? Let me look... Okay, the endpoint is UDP port 53133 on host 216.58.211.110.
* Send encrypted bytes from step 2 over the Internet to 216.58.211.110:53133 using UDP.

When the interface receivesa packet, this happens:
* I just got a packet from UDP port 7361 on host 98.139.183.24. Let's decrypt it!
* It decrypted and authenticated properly for peer LMNOPQRS. Okay, let's remember that peer LMNOPQRS's most recent Internet endpoint is 98.139.183.24:7361 using UDP.
* Once decrypted, the plain-text packet is from 192.168.43.89. Is peer LMNOPQRS allowed to be sending us packets as 192.168.43.89?
* If so, accept the packet on the interface. If not, drop it.

Wireguard uses the public key to uniquely identify and route a client. This means that **you can't have the same key on two clients that are simultaneously connected to the same server**.

# Installing
## OpenWRT
* Navigate to LuCI-System-Software and install the packages
  * luci-proto-wireguard
  * qrencode (so openwrt can generate QRcode for client config)
* Generating keys 
  * Generate a key pair of private and public keys.
  * `wg genkey | tee wg.key | wg pubkey > wg.pub`
  * Use the wg.key file to configure the WireGuard interface on this router.
  * Use the wg.pub file to configure peers that will connect to this router through the WireGuard VPN.
  * restart network (can be done via luci-system-startup-initscript-network-restart), but easiest is via CLI `/etc/init.d/network restart'
  * setting up network
    * To create a new WireGuard interface go to LuCI Network Interfaces Add new interface... and select WireGuard VPN from the Protocol dropdown menu.
    * select the keys generated in step2 above
    * IP addresses: 10.0.0.1/32 (the IP address of the wireguard interface)
    * monitoring status: either via luci-status-wireguard, or CLI `wg`. The wg command should give the wg interface, and all peers that have completed a succesfull handshake (exchange of private/public keys).

## Client
I have done this on iphone (IOS 17.3) and Android (13).
* install the wireguard app
* on openwrt 
  * goto luci-interfaces-wireguard and select tab 'peers'.
  * click on 'add peer' and fill in 
    * name; 
    * click 'generate new key pair', 
    * for Allowed IPs fill in '10.0.0.2/32'; this is the IP address of the client
    * Route Allowed IPs: yes.
    * endpoint host: the url of your home, or the **external** IP address of your ISP router (if not known, google 'find my ip address'
    * endpoint port: 51820 
    * keepalive: 25
    * now it should be possible to click on 'generate configuration' QR-code
* on cient
  * add new tunnel by clicking on '+' sign, and scan the QR code
  * edit the new tunnel to check the settings
    * set DNS to 9.9.9.9
    * set endpoint to `<your public ip address>:51820`
* on ISP modem
    * ensure that port 51820 is forwarded to Openwrt, or put Openwrt in the DMZ of your ISP router
* on openwrt
  * luci-network-firewall, select tab traffic rules; add rule
    * name: wireguard
    * protocol: UDP (wg uses UDP)
    * source zone: WAN (packets originate from outside world)
    * source address: any IP (the IP of the client is not known)
    * source port: any (also not known)
    * destination zone: Device (input); the packet should be handled by wg on openwrt
    * destination address: add IP (not filled in)
    * destination port: 51820 (we use this default port for wg)
    * action: accept
  * luci-network-firewall, tab 'general settings'
    * add zone, call it 'wireguard'
      * input: accept
      * output: accept
      * forward: reject
      * masquerading: off
      * covered networks: wireguard
      * allow forward to destination: LAN, WAN (so you can access via wg both your LAN and internet)
      * allow forward from source zones: unspecified
      * Masquerading in only needed for WAN zone, not for LAN zone, as specified at some sites; I do not understand why it is not needed for LAN zone, but I can still access the LAN via wg.

Now it is good to check whether the settings made via Luci, ended up correctly in the firewall and network settings files.
So `cat /etc/config/network` the relevant part should look like this:
```
config interface 'wireguard'
        option proto 'wireguard'
        option private_key 'REDACTED'
        option listen_port '51820'
        list addresses '10.0.0.1/24'

config wireguard_wireguard
        option public_key 'REDACTED'
        option description 'try1'
        list allowed_ips '10.0.0.2/32'
        option route_allowed_ips '1'
        option endpoint_host 'your_publicIP_or name'
        option persistent_keepalive '25'
```
If you want to add more peers, then each peer must have a unique IP-address; So the next peer could have address `10.0.0.03/32`.

Note: /32 indicates exactly one IP-address (/24 indicates a range of 255 IP addresses)

# Testing an troubleshooting
To test, first turn off wifi on you phone, so we know for sure that traffic is not floating via your wifi. Of course mobile data must be turned on.

Activate the connection and try whether you can reach your LAN and internet sites.

On the command prompt of your openwrt you can give the command `wg` to see whether handshaking is succesful or not. This is the first part that must be working. If this is not working double check the keys on openwrt and your phone.

## Check whether traffic arrives at your openwrt
When these are OK, we are first going to check whether traffic is arriving at openwrt from your phone.
First chech on openwrt CLI, whehter port 51820 is listened to by `netstat -nulp`, this shoudl list at least port 51820. If wg is implemented as a kernel module, you do not see a PID/program name after 51820.

Now we know that wireguard is listening to this UDP port, we check further. It does sound strange but we can use `tcpdump` to monitor also UDP packets. Use `tcpdump -i wan udp port 51820` and try to make a connection from your phone. Now you should see some packets arriving at the wan interface op openwrt. If not, then something is wrong with forwarding from your ISP router, of you client/phone is not working correctly.



