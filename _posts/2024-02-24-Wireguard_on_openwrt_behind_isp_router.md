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
* 2. Generating keys @@@

    Generate a key pair of private and public keys.
        wg genkey | tee wg.key | wg pubkey > wg.pub
            *
            Use the wg.key file to configure the WireGuard interface on this router.
            *
            Use the wg.pub file to configure peers that will connect to this router through the WireGuard VPN.
    staan in root


