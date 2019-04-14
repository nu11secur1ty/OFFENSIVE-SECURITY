***Kali Rolling ISO of DOOM, Too.***

A while back we introduced the idea of Kali Linux Customisation by demonstrating the Kali Linux ISO of Doom. Our scenario covered the installation of a custom Kali configuration that contained select tools required for a remote vulnerability assessment. The customised Kali ISO would undergo an unattended autoinstall in a remote client site and automatically connect back to our OpenVPN server over TCP port 443. The OpenVPN connection would then bridge the remote and local networks, allowing us full “layer 3” access to the internal network from our remote location. The resulting custom ISO could then be sent to the client who would just pop it into a virtual machine template and the whole setup would happen automagically with no intervention – as depicted in the image below.
![](https://github.com/nu11secur1ty/OFFENSIVE-SECURITY/blob/master/Setting%20Up%20the%20OpenVPN%20Server/img/kali-linux-agent2.png)

***Setting Up the OpenVPN Server***


Before we begin, we’ll set up the OpenVPN server – which has a public IP address (a.b.c.d). This is also the server on which we will build and generate the custom agent ISO file. On the remote Kali server, create and configure OpenVPN as follows:

```bash
apt-get install openvpn
cd /root/
cp -rf /usr/share/easy-rsa/ test
cd test
cp keys/{server.crt,server.key,dh2048.pem,ca.crt} /etc/openvpn/
cd /etc/openvpn

cat << EOF > server.conf
tls-server
port 443
proto tcp
dev tap
ca ca.crt
cert server.crt
key server.key # This file should be kept secret
dh dh2048.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
client-config-dir static
keepalive 10 120
comp-lzo
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
EOF

mkdir -p static

cat << EOF > static/client
ifconfig-push 10.8.0.200 255.255.255.0
EOF
```
Next, we create SSH keys, enable IP forwarding on the VPN server and configure iptables. Once that’s done, we start the OpenVPN server.

```bash
ssh-keygen
openvpn --cd /etc/openvpn --config /etc/openvpn/server.conf
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
