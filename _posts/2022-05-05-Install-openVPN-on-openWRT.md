---
layout: post
title: Install OpenVPN on router running OpenWRT
---

In this blog I explain how you can install and configure openVPN on your Openwrt router.

# Introduction
## Goal
I use OpenVPN to access my LAN services when I am not at home (for instance from my holiday address), without using unsecure port forwarding for all services.

With OpenVPN I have only one 'hole' in my firewall, that can only be accessed when you have the right key.

## Prerequisites
* You should have a router with OpenWRT already installed, and you should have access via SSH to this OpenWRT router.

I have tested the instructions on my Linksys WRT3200ACM with OpenWRT 21.01.

## Used sources
I have not invented all instructions myself :), but have used this site:
* https://openwrt.org/docs/guide-user/services/vpn/openvpn/server
I have added extra instructions where necessary, and also added comments on why certain instructions are necessary.

# Instructions
NB: I used the above site, but following the instructions from that site exactly resulted in a non operating router, so please stick to the instructions below.

## End situation and inner workings
First I describe the end situation, and how openVPN will work after you have followed this guide. This to make you understand what is happening.

You will have 2 instances of OpenVPN running:
* servertcp443.conf
* serverudp1194.conf

The first one is listening on TCP port 443. This port is always open at every location you visit (it is used by https); I made this instance because the default OpenVPN UDP port 1194 is not always usable when you are traveling.
Just to be sure I also added an instance that listens to UDP port 1194.

After you have installed the OpenVPN package these two instances will be automatically started when the above mentioned files are present in directory `/etc/openvpn/

NB: this guide does not use the GUI method (Luci) for installing OpenVPN, but uses command line. When you should configure OpenVPN via Luci the config files are placed in `/etc/config/openvpn`. `/etc/init.d/x` reads `/etc/config/x` and OVERwrites to `/etc/x/`. So you should not mix configuring via Luci and via command line!


Both OpenVPN instances have their own
* tun device
* subset (of IP-addresses that are assigned to clients)

Of course in the firewall rules are added to accept the traffic sent by a client to one of the two openVPN instances.

### Incoming connection from WAN
* If the VPN client sends a connection setup to your ISP's modem, that modem should forward this to your OpenWRT router. Normally this is done by assigning the role as DMZ server to the OpenVPN router, so no port forwarding is necessary for this.
* udp/1194 arrives from  wan@openwrt - and is ACCEPTed by firewall in OpenWRT
* the OpenVPN process assigns IP 192.168.8.x to the VPN client
* zone 'vpn' is defined with subnet 192.168.8.0/24' (later on also 192.168.9.0/24 added for the 2nd instance of OpenVPN)
* this subnet makes the incoming traffic to arrive in zone 'vpn' (look at network->firewall; edit zone vpn->advanced settings: there you see "Use this option to classify zone traffic by source or destination subnet instead of networks or devices" and there is 192.168.8.0 filled in. Probably it is also possible to use device tun0 to classify the traffic in stead of the subnet(s).
* lan zone is allowed to forward traffic to the vpn zone (fw settings->general)
* vpn zone is allowed to forward traffic to lan zone
* so the vpn traffic can travel to all destinations (because lan is allowed to forward traffic to wan)


For the client side you get two .ovpn files (one for each OpenVPN instance) that can be imported in the client OpenVPN application.

*NB: later on I decided to disable incoming traffic on UPD/1194, because there were too many attacks from foreign IP addresses. This guide however still contains the instructions for both instances of the OpenVPN server*


## Install packages
Install the required packages. 

```
opkg update
opkg install openvpn-openssl openvpn-easy-rsa
#Configuration parameters
#OVPN_POOL config any network are OK except your local network\
OVPN_DIR="/etc/openvpn"
OVPN_PKI="/etc/easy-rsa/pki"
OVPN_PORT="1194"
OVPN_PROTO="udp"
OVPN_POOL="192.168.8.0 255.255.255.0"
OVPN_DNS="${OVPN_POOL%.* *}.1"
OVPN_DOMAIN="$(uci get dhcp.@dnsmasq[0].domain)"
 
#Fetch WAN IP address
. /lib/functions/network.sh
network_flush_cache
network_find_wan NET_IF
network_get_ipaddr NET_ADDR "${NET_IF}"
OVPN_SERV="${NET_ADDR}"
 
#Fetch FQDN from DDNS client
NET_FQDN="$(uci -q get ddns.@service[0].lookup_host)"
if [ -n "${NET_FQDN}" ]
then OVPN_SERV="${NET_FQDN}"
fi
```

## Key management

Use EasyRSA to manage the PKI. Utilize private key password protection if necessary.

NB: generating keys can take like 10 minutes.

```
# Configuration parameters
export EASYRSA_PKI="${OVPN_PKI}"
export EASYRSA_REQ_CN="ovpnca"
export EASYRSA_BATCH="1"
export EASYRSA_CERT_EXPIRE="3650" # Increases the client cert expiry from the default of 825 days to match the CA expiry
 
#Remove and re-initialize PKI directory\
easyrsa init-pki
 
#Generate DH parameters\
easyrsa gen-dh
 
#Create a new CA\
easyrsa build-ca nopass
 
#Generate server keys and certificate\
easyrsa build-server-full server nopass
openvpn --genkey tls-crypt-v2-server ${EASYRSA_PKI}/private/server.pem
 
#Generate client keys and certificate\
easyrsa build-client-full client nopass
openvpn --tls-crypt-v2 ${EASYRSA_PKI}/private/server.pem \
--genkey tls-crypt-v2-client ${EASYRSA_PKI}/private/client.pem
```

## FIREWALL
The firewall configuration file at `/etc/config/firewall` should have following contents:
```
config defaults
	option syn_flood '1'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'REJECT'

config zone
	option name 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'
	list network 'lan'
	list network 'vpn'
	list network 'tun1'
#put lan, vpn and tun1 in same zone

config zone
	option name 'wan'
	list network 'wan'
	list network 'wan6'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'

config zone
	option name 'vpn'
	option input 'ACCEPT'
	option forward 'REJECT'
	option output 'ACCEPT'
	option mtu_fix '1'
	list subnet '192.168.8.0/24'
	list subnet '192.168.9.0/24'
#each instance of the OpenVPN server has its own subnet to give out IP-addresses

config forwarding
	option dest 'lan'
	option src 'vpn'

config forwarding
	option dest 'vpn'
	option src 'lan'
#allow vpn trafic to go to lan and vice versa

config forwarding
	option src 'lan'
	option dest 'wan'

config rule
	option name 'Allow-DHCP-Renew'
	option src 'wan'
	option proto 'udp'
	option dest_port '68'
	option target 'ACCEPT'
	option family 'ipv4'

config rule
	option name 'Allow-Ping'
	option src 'wan'
	option proto 'icmp'
	option icmp_type 'echo-request'
	option family 'ipv4'
	option target 'ACCEPT'

config rule
	option name 'Allow-IGMP'
	option src 'wan'
	option proto 'igmp'
	option family 'ipv4'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCPv6'
	option src 'wan'
	option proto 'udp'
	option src_ip 'fc00::/6'
	option dest_ip 'fc00::/6'
	option dest_port '546'
	option family 'ipv6'
	option target 'ACCEPT'

config rule
	option name 'Allow-MLD'
	option src 'wan'
	option proto 'icmp'
	option src_ip 'fe80::/10'
	list icmp_type '130/0'
	list icmp_type '131/0'
	list icmp_type '132/0'
	list icmp_type '143/0'
	option family 'ipv6'
	option target 'ACCEPT'

config rule
	option name 'Allow-ICMPv6-Input'
	option src 'wan'
	option proto 'icmp'
	list icmp_type 'echo-request'
	list icmp_type 'echo-reply'
	list icmp_type 'destination-unreachable'
	list icmp_type 'packet-too-big'
	list icmp_type 'time-exceeded'
	list icmp_type 'bad-header'
	list icmp_type 'unknown-header-type'
	list icmp_type 'router-solicitation'
	list icmp_type 'neighbour-solicitation'
	list icmp_type 'router-advertisement'
	list icmp_type 'neighbour-advertisement'
	option limit '1000/sec'
	option family 'ipv6'
	option target 'ACCEPT'

config rule
	option name 'Allow-ICMPv6-Forward'
	option src 'wan'
	option dest '*'
	option proto 'icmp'
	list icmp_type 'echo-request'
	list icmp_type 'echo-reply'
	list icmp_type 'destination-unreachable'
	list icmp_type 'packet-too-big'
	list icmp_type 'time-exceeded'
	list icmp_type 'bad-header'
	list icmp_type 'unknown-header-type'
	option limit '1000/sec'
	option family 'ipv6'
	option target 'ACCEPT'

config rule
	option name 'Allow-IPSec-ESP'
	option src 'wan'
	option dest 'lan'
	option proto 'esp'
	option target 'ACCEPT'

config rule
	option name 'Allow-ISAKMP'
	option src 'wan'
	option dest 'lan'
	option dest_port '500'
	option proto 'udp'
	option target 'ACCEPT'

config rule
	option src 'wan'
	option proto 'udp'
	option dest_port '1194'
	option family 'ipv4'
	option target 'ACCEPT'
	option name 'OpenVPNupd1194'
#this is the rule to accept VPN traffic at UDP/1194
#NB: due to high amounts of attack traffic at this port I have changed ACCEPT into REJECT

config rule
	option name 'OpenVPNtcp443'
	option src 'wan'
	option target 'ACCEPT'
	list proto 'tcp'
	option dest_port '443'
	option family 'ipv4'

config rule
	option name 'Support-UDP-Traceroute'
	option src 'wan'
	option dest_port '33434:33689'
	option proto 'udp'
	option family 'ipv4'
	option target 'REJECT'
	option enabled '0'

config include
	option path '/etc/firewall.user'
```


## VPN service

Configure VPN service and generate client profiles.

```
# Configure VPN service and generate client profiles
umask go=
OVPN_DH="$(cat ${OVPN_PKI}/dh.pem)"
OVPN_CA="$(openssl x509 -in ${OVPN_PKI}/ca.crt)"
ls ${OVPN_PKI}/issued \
| sed -e "s/\.\w*$//" \
| while read -r OVPN_ID
do
OVPN_TC="$(cat ${OVPN_PKI}/private/${OVPN_ID}.pem)"
OVPN_KEY="$(cat ${OVPN_PKI}/private/${OVPN_ID}.key)"
OVPN_CERT="$(openssl x509 -in ${OVPN_PKI}/issued/${OVPN_ID}.crt)"
OVPN_EKU="$(echo "${OVPN_CERT}" | openssl x509 -noout -purpose)"
case ${OVPN_EKU} in
(*"SSL server : Yes"*)
OVPN_CONF="${OVPN_DIR}/${OVPN_ID}.conf"
cat << EOF > ${OVPN_CONF} ;;
user nobody
group nogroup
dev tun
port ${OVPN_PORT}
proto ${OVPN_PROTO}
server ${OVPN_POOL}
topology subnet
client-to-client
keepalive 10 60
persist-tun
persist-key
push "dhcp-option DNS ${OVPN_DNS}"
push "dhcp-option DOMAIN ${OVPN_DOMAIN}"
push "redirect-gateway def1"
push "persist-tun"
push "persist-key"
<dh>
${OVPN_DH}
</dh>
EOF
(*"SSL client : Yes"*)
OVPN_CONF="${OVPN_DIR}/${OVPN_ID}.ovpn"
cat << EOF > ${OVPN_CONF} ;;
user nobody
group nogroup
dev tun
nobind
client
remote ${OVPN_SERV} ${OVPN_PORT} ${OVPN_PROTO}
auth-nocache
remote-cert-tls server
EOF
esac
cat << EOF >> ${OVPN_CONF}
<tls-crypt-v2>
${OVPN_TC}
</tls-crypt-v2>
<key>
${OVPN_KEY}
</key>
<cert>
${OVPN_CERT}
</cert>
<ca>
${OVPN_CA}
</ca>
EOF
done
/etc/init.d/openvpn restart
ls ${OVPN_DIR}/*.ovpn
```

This script generates only the .ovpn file the UDP/1194. You should copy this file and change a number of things to create a server config file for the TCP/443 version. You have to adapt the following after copying:
```
user nobody
group nogroup
dev tun0
port 1194
proto udp
server 192.168.8.0 255.255.255.0
topology subnet
client-to-client
keepalive 10 60
persist-tun
persist-key
push "dhcp-option DNS 192.168.8.1"
push "dhcp-option DOMAIN lan"
push "redirect-gateway def1"
push "persist-tun"
push "persist-key"
```
to
```
user nobody
group nogroup
dev tun1
#listen to this interface only; not all interfaces; otherwise
#local 192.168.2.253
port 443
proto tcp
server 192.168.9.0 255.255.255.0
topology subnet
client-to-client
keepalive 10 60
persist-tun
persist-key
push "dhcp-option DNS 192.168.9.1"
push "dhcp-option DOMAIN lan"
push "redirect-gateway def1"
push "persist-tun"
push "persist-key"

```
The rest of the file should not be changed!

@@@Perform OpenWrt backup. Extract client profiles from the archive and import them to your clients.

If you want additional clients .ovpn run this multi-client script:
```
# Configuration parameters
OVPN_DIR="/etc/openvpn"
OVPN_PKI="/etc/easy-rsa/pki"
OVPN_PORT="1194"
OVPN_PROTO="udp"
OVPN_POOL="192.168.8.0 255.255.255.0"
OVPN_DNS="${OVPN_POOL%.* *}.1"
OVPN_DOMAIN="$(uci get dhcp.@dnsmasq[0].domain)"
 
 
# Fetch FQDN from DDNS client
NET_FQDN="$(uci -q get ddns.@service[0].lookup_host)"
if [ -n "${NET_FQDN}" ]
then OVPN_SERV="${NET_FQDN}"
fi
# Configuration parameters
export EASYRSA_PKI="${OVPN_PKI}"
export EASYRSA_REQ_CN="ovpnca"
export EASYRSA_BATCH="1"
 
 
# Generate client keys and certificate
easyrsa build-client-full client3 nopass
openvpn --tls-crypt-v2 ${EASYRSA_PKI}/private/server.pem \
--genkey tls-crypt-v2-client ${EASYRSA_PKI}/private/client.pem
```


```
 
# Revoke client certificate
easyrsa revoke client
 
 
/etc/init.d/openvpn restart
```




Now make a script consisting of the “Configuration parameters” of Part 1 above and all of Part 4 above and run it. Note that the “remote” line may be missing in the new ovpn (use the original as a reference for that).

 
