---
layout: post
title: Wireguard VPN on openWRT router behind ISP router
---
Used sources
* https://forum.openwrt.org/t/wireguard-server-on-openwrt-router-behind-isp-router-firewall-config/189027/20
* https://openwrt.org/docs/guide-user/services/vpn/wireguard/basics
* My own experience to get this working :)

# Goal and introduction
If I am away form home, for instance on holiday, I want to have access to my LAN, including all equipment attached to the LAN.
I was using OpenVPN on OpenWRT for that. That was working.
But after upgrading OpenWRt I lost the configuration of OpenVPN, and I knew it was a big hassle to get it working again.

I read somewhere that *Wireguard* should be better and simpler. So I decided to give Wireguard a try.

The configuraion appeared to be not that simple, at least not if you are not a network expert.

I tried for several days to get it working myself, but get stuck. So finally I asked for help in the OpenWRT forum. And there are a lot of experts willing to help, and I got in working within one day.
This post describes what I have done. I hope it will be helpfull for others.


# Situation
Equipment used
* OpenWrt 23.05.0 on Linksys WRT3200ACM

Configuration
* ISP router
  * 192.168.2.254

* OpenWRT router
  * 192.168.2.253 (WAN interface)
  * 192.168.1.1 (internally)
  * 192.168.1.0/24 LAN network


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

# Installing and configuring
## OpenWRT
* Navigate to LuCI-System-Software and install the packages
  * luci-proto-wireguard
  * qrencode (so openwrt can generate QRcode for client config)
* Generating keys 
  * Generate a key pair of private and public keys.
  * `wg genkey | tee wg.key | wg pubkey > wg.pub`
  * Use the wg.key file to configure the WireGuard interface on this router.
  * Use the wg.pub file to configure peers that will connect to this router through the WireGuard VPN.
  * Restart network (can be done via luci-system-startup-initscript-network-restart), but easiest is via CLI `/etc/init.d/network restart'
  * Setting up network
    * To create a new WireGuard interface go to LuCI Network Interfaces Add new interface... and select WireGuard VPN from the Protocol dropdown menu.
    * select the keys generated in step2 above
    * IP addresses: 10.0.0.1/32 (the IP address of the wireguard interface)
    * Monitoring status: either via luci-status-wireguard, or CLI `wg`. The wg command should give the wg interface, and all peers that have completed a succesfull handshake (exchange of private/public keys).

* On ISP modem
    * Ensure that port 51820 (the default port used by Wireguard) is forwarded to OpenWRT, or put Openwrt in the DMZ of your ISP router
* On openwrt
  * luci-network-firewall, select tab traffic rules; add rule
    * name: wireguard
    * protocol: UDP (wg uses UDP)
    * source zone: WAN (packets originate from outside world)
    * source address: any IP (the IP of the client is not known)
    * source port: any (also not known)
    * destination zone: Device (input); the packet should be handled by wg on OpenWRT
    * destination address: (leave empty)
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
      * Masquerading in only needed for WAN zone, not for LAN zone, see also below.

Now it is good to check whether the settings made via Luci, ended up correctly in the firewall and network settings files.
So `cat /etc/config/network` the relevant part should look like this:
```
config interface 'wireguard'
        option proto 'wireguard'
        option private_key 'REDACTED'
        option listen_port '51820'
        list addresses '10.0.0.1/24'
```

The relevant part of /etc/config/firewall should look like this:
```
config zone
        option name 'lan'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'ACCEPT'
        list network 'lan'

config rule
        option name 'wireguard'
        option src 'wan'
        option dest_port '51820'
        option target 'ACCEPT'
        list proto 'udp'

config zone
        option name 'wireguard'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'REJECT'
        list network 'wireguard'

config forwarding
        option src 'wireguard'
        option dest 'lan'

config forwarding
        option src 'wireguard'
        option dest 'wan'

```

### Masquerading is not necessary on the LAN zone!
Some sites suggest that you should activate masquerading (NATting) on the LAN-zone. This seams not to be necessary, at least not in this configuration.
I can reach my raspberry pi on my lan, via wireguard no my phone (with wifi turned off), without masquerading on the LAN zone.
Tcpdump shows that packets from `10.0.0.2` (IP address of the wg tunnel on my phone) on my raspberry pi5 (named rpi5) which has an IP address of `192.168.1.15`. And that my rpi5 is ending packets back to `10.0.0.2`. I assume that this is possible because openwrt/wg knows to find my rpi5, and rpi5 has openwrt as default gateway, and openwrt/wg knows how to find 10.0.0.2. See also the output of `tcpdump -vv -i end0 host 10.0.0.2` executed on my rpi5 below (end0 is the name of the ethernet interface of my rpi5).

```
20:04:56.166333 IP (tos 0x0, ttl 63, id 0, offset 0, flags [DF], proto TCP (6), length 64)
    **10.0.0.2.49267** > rpi5.ssh: Flags [S], cksum 0x6ef0 (correct), seq 1516598280, win 65535, options [mss 1220,nop,wscale 6,nop,nop,
20:04:56.166388 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    rpi5.ssh > **10.0.0.2.49267**: Flags [S.], cksum 0xcbe7 (incorrect -> 0x57e8), seq 3583316629, ack 1516598281, win 31856, options [me 7], length 0

```

NB: you can also edit the /etc/config/firewall and network files directly, in stead of via Luci. But bear in mind to always restart the network and firewall (via `/etc/init.d/network restart` or `/etc/init.d/firewall restart`, or reboot openWRT router.



## Client on IOS and Android
I have done this on iphone (IOS 17.3) and Android (13).
* Install the Wireguard app
* On OpenWRT
  * goto luci-interfaces-wireguard and select tab 'peers'.
  * click on 'add peer' and fill in 
    * name; 
    * click 'generate new key pair', 
    * for Allowed IPs fill in '10.0.0.2/32'; this is the IP address of the client; **do not fill in here the IP-range of the subnet that you want to be able to reach from remote location; this address range needs only be filled in on the client config (see below)**
    * Route Allowed IPs: yes.
    * endpoint host: the url of your home, or the **external** IP address of your ISP router (if not known, google 'what is my ip address'
    * endpoint port: 51820 
    * keepalive: 25 (not necessary)
    * now it should be possible to click on 'generate configuration' QR-code
* On client
  * add new tunnel by clicking on '+' sign, and scan the QR code
  * edit the new tunnel to check the settings
    * set DNS to 9.9.9.9
    * set AllowedIPs to 192.168.1.0/24 #the address range of the subnet that should be reachable via wg. If you want all traffic to be routed via wg, the fill in `0.0.0.0/0`  for IPv4.
    * set endpoint to `<your public ip address, or name>:51820`

If you want to add more peers, then each peer must have a unique IP-address; So the next peer could have address `10.0.0.03/32`. After you added a new client following the above procedure, and assigning a unique IP-address, you **have to restart the network** `/etc/init.d/network restart`, then activate the connection at the client, and check on openwrt via `wg` whether you see the newly added client.

**NB: you MUST restart the network (for instructions see above) after adding a new peer (client), otherwise the peer will not get a handshake!**


Note: /32 indicates exactly one IP-address (/24 indicates a range of 255 IP addresses)

The relevant part of `/etc/config/network` should look like this:
```
config wireguard_wireguard
        option public_key 'REDACTED'
        option description 'try1'
        list allowed_ips '10.0.0.2/32'
        option route_allowed_ips '1'
        option endpoint_host 'your_publicIP_or name'
        option persistent_keepalive '25'
```



## Client on Linux
I have done this on Linux Mint, based on Ubuntu 22.

* on Linux
  * install wireguard via `apt install wireguard`
  * generate key pair
    * `wg genkey | tee private.key | wg pubkey > public.key`
    * Use the pubic key to configure the WireGuard peer on OpenWRT (see instructions above for adding IOS or Android peer)
    * the private key is used below
  * execute following
```cat <<EOF >/etc/wireguard/wg0.conf
[Interface]
PrivateKey = private key generated for this peer
Address = 10.0.0.6/32
[Peer]
PublicKey = public key of your wireguard server on openwrt
AllowedIPs = 192.168.1.0/24
Endpoint = <public IP address of your ISP modem>:51820
EOF
```

Address must be a unique IP address that will be assigned to this peer.
AllowedIPs contain a list of IP addresses/ranges that must be able to transfer through the wt tunnel. So if you want to be able to reacht your LAN with 192.168.1.0/24, this must be part of the AllowedIPs.

* on Linux client
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
      * Masquerading in only needed for WAN zone, not for LAN zone, see also below.

Now `/etc/wireguard/wg0.conf` on your linux client should look like this:
```
[Interface]
PrivateKey = REDACTED
Address = 10.0.0.6/32
[Peer]
PublicKey = REDACTED
AllowedIPs = 192.168.1.0/24
Endpoint = <your public IP>:51820
```

Now you can activat wg via `wg-quick up wg0` 
This command also adds some routing rules. These can be viewed via 'ip route'.
To check the status of wg issue command `wg`. This should specify among others the time of the latest handshake.


**NB: If you want to direct all trafic through the wg tunnel, by specifyin `Address = 0.0.0.0/0` then the 'wg-quick up wg0` command will use Policy Based Routing in linux and creates a new routing table. It also adds a routing rule then specifies a higher priority for this table than the main table has. This can be viewed via `ip rule`.

## Client on Windows
@@@

# Testing an troubleshooting
To test, first turn off wifi on your phone, so we know for sure that traffic is not floating via your local wifi. Of course mobile data must be turned on.

Activate the connection and try whether you can reach your LAN and internet sites.

On the command prompt of your openwrt you can give the command `wg` to see whether handshaking is succesful or not. This is the first part that must be working. If this is not working double check the keys on openwrt and your phone, and restart the network on Openwrt via `/etc/init.d/network restart`.

## Check whether traffic arrives at your openwrt
When these are OK, we are first going to check whether traffic is arriving at openwrt from your phone.
First chech on openwrt CLI, whehter port 51820 is listened to by `netstat -nulp`, this shoudl list at least port 51820. If wg is implemented as a kernel module, you do not see a PID/program name after 51820.

Now we know that wireguard is listening to this UDP port, we check further. It does sound strange but we can use `tcpdump` to monitor also UDP packets. Use `tcpdump -i wan udp port 51820` and try to make a connection from your phone. Now you should see some packets arriving at the wan interface op openwrt. If not, then something is wrong with forwarding from your ISP router, of you client/phone is not working correctly.

Let's first check if we generate UDP traffic on port 51820 ourselves, whether that UDP traffice arrives at the WAN interface of openwrt.

Generate test UDP packets. Open a second ssh session to you openwrt, or any other linux box you have and give following command `nc -u <your public ip addr> 51820 < /dev/random`. `nc` stands for netcat and must be installed if not available. At the same time you give the following command in het openwrt shell `tcpdump -i wan udp port 51820` and you should see all kind of traffic being listed. If not then this UDP port is not probably not forwarded by your ISP router.
If you get 'connection refused' then the packet is refused probably
You can test this by generating traffic at UDP port 53 (used for DNS queries), this port must be forwarded to openwrt `nc -u <your public ip addr> 53 < /dev/random`.

When everything is OK you should see with `tcpdump` traffic arriving at both the `wan` interface as well as at the `wireguard` interface.

## firewall problems
If the wg traffic is arriving at the wan interface, but you cannot access your LAN or internet sites, then double check your firewall settings.
