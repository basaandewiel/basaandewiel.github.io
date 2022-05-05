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

I have tested this on Linksys WRT3200ACM with OpenWRT 21.01.

## Used sources
I have not invented all instructions myself :), but have used this site:
* https://openwrt.org/docs/guide-user/services/vpn/openvpn/server


# Instructions
NB: I used the above site, but following the instructions from that site exactly resulted in a non operating router, so please stick to the instructions below.

## End situation
Firs I describe the end situation, and how openVPN will work after you have followed this guide. This to maken you understand what is happening.

You will have 2 instances of OpenVPN running:
* servertcp443.conf
* serverudp1194.conf

The first one is listening on TCP port 443. This port is always open (it is used by https); I made this instance because the default UDP port 1194 is not always open when you are traveling.
Just to be sure I also added an instance that listens to UDP port 1194.

After you have installed the OpenVPN package these two instances will be automatically started when the above mentioned files are present in directory `/etc/openvpn/

NB: this guide does not use the GUI methode (Luci) for installing OpenVPN, but uses command line.

Both OpenVPN instances you their own
* tun device
* subset (of IP-addresses that are assigned to clients)




## Install packages
Install the required packages. Specify the VPN server configuration parameters.

```opkg update
opkg install openvpn-openssl openvpn-easy-rsa
# Configuration parameters
# OVPN_POOL config any network are OK except your local network\
OVPN_DIR="/etc/openvpn"
OVPN_PKI="/etc/easy-rsa/pki"
OVPN_PORT="1194"
OVPN_PROTO="udp"
OVPN_POOL="192.168.8.0 255.255.255.0"
OVPN_DNS="${OVPN_POOL%.* *}.1"
OVPN_DOMAIN="$(uci get dhcp.@dnsmasq[0].domain)"
 
# Fetch WAN IP address
. /lib/functions/network.sh
network_flush_cache
network_find_wan NET_IF
network_get_ipaddr NET_ADDR "${NET_IF}"
OVPN_SERV="${NET_ADDR}"
 
# Fetch FQDN from DDNS client
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

@@@FIREWALL

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

Perform OpenWrt backup. Extract client profiles from the archive and import them to your clients.

If you want additional clients .ovpn run this multi-client script:
```
# Configuration parameters
OVPN_PKI="/etc/easy-rsa/pki"
export EASYRSA_PKI="${OVPN_PKI}"
export EASYRSA_BATCH="1"
 
# Add one more client
easyrsa build-client-full client1 nopass
openvpn --tls-crypt-v2 ${EASYRSA_PKI}/private/server.pem \
--genkey tls-crypt-v2-client ${EASYRSA_PKI}/private/client1.pem
 
# Revoke client certificate
easyrsa revoke client
 
# Generate a CRL
easyrsa gen-crl
 
# Enable CRL verification
OVPN_CRL="$(cat ${OVPN_PKI}/crl.pem)"
cat << EOF >> /etc/openvpn/server.conf
<crl-verify>
${OVPN_CRL}
</crl-verify>
EOF
/etc/init.d/openvpn restart
```




Now make a script consisting of the “Configuration parameters” of Part 1 above and all of Part 4 above and run it. Note that the “remote” line may be missing in the new ovpn (use the original as a reference for that).

 
