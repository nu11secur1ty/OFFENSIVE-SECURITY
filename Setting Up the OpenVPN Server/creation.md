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

***Building the Custom Kali Rolling ISO***

Now, we create the custom ISO which will autoinstall itself and auto-connect back with the OpenVPN configuration we set it with. Other than starting the installation via the ISO, nothing else will be needed for this agent to connect back to us. We start by installing live-build prerequisites and modifying the default configuration to include only our required packages and configuration files.

```bash
apt-get install git live-build cdebootstrap
git clone git://git.kali.org/live-build-config.git build
cd build
```

We then define the packages we would like installed in the Agent ISO. In this case, it’s a minimal subset of tools as well as the required SSH and OpenVPN daemons:

```bash
echo nmap > kali-config/variant-default/package-lists/kali.list.chroot
echo openssh-server >> kali-config/variant-default/package-lists/kali.list.chroot
echo openvpn >> kali-config/variant-default/package-lists/kali.list.chroot
echo metasploit-framework >> kali-config/variant-default/package-lists/kali.list.chroot
```

Once done, we copy over the OpenSSH and OpenVPN keys and configuration files on to the chroot overlay:

```bash
mkdir -p kali-config/common/includes.chroot/etc/openvpn
cp /root/test/keys/{ca.crt,client.crt,client.key} kali-config/common/includes.chroot/etc/openvpn/

mkdir -p kali-config/common/includes.chroot/root/.ssh/
cp /root/.ssh/id_rsa.pub kali-config/common/includes.chroot/root/.ssh/authorized_keys

# Create the OpenVPN client configuration file
cat << EOF > kali-config/common/includes.chroot/etc/openvpn/client.conf
client
dev tap
proto tcp
remote a.b.c.d 443 # remote server IP
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
ns-cert-type server
comp-lzo
verb 3
EOF
```
We introduce a new boot option which will process the preseed file, and boot this choice with no user interaction:

```bash
cat << EOF > kali-config/common/includes.binary/isolinux/install.cfg
label install
menu label ^Install
linux /install/vmlinuz
initrd /install/initrd.gz
append vga=788 -- quiet file=/cdrom/install/preseed.cfg locale=en_US keymap=us hostname=kali domain=local.lan
EOF

cat << EOF > kali-config/common/includes.binary/isolinux/isolinux.cfg
include menu.cfg
ui vesamenu.c32
default install
prompt 0
timeout 5
EOF
```
And create a couple of hooks to have the SSH and OpenVPN services start at boot time:

```bash
echo 'update-rc.d -f ssh enable' > kali-config/common/hooks/01-start-ssh.chroot
echo 'update-rc.d -f openvpn enable' > kali-config/common/hooks/01-start-openvpn.chroot
chmod +x kali-config/common/hooks/*.chroot
```

Lastly, we include a preseed file which will skip all the common installation questions.


```bash
mkdir -p kali-config/common/includes.installer
wget https://www.kali.org/dojo/preseed.cfg -O ./kali-config/common/includes.installer/preseed.cfg
```


Now that all our ducks are in a row, we generate the agent ISO file. Once the file is ready, we send it over to be installed in the local network we want bridged to us.

```bash
./build.sh --distribution kali-rolling --verbose
```
***Bridging the Network Gaps***

Once the VPN connection is established by the client, we can SSH to our internal Kali Linux agent and complete the final requirement: to bridge the remote and local networks together.

- On the Server

We enable routing to the remote network on the OpenVPN server:

```bash
root@kali:~# route add -net 192.168.101.0/24 gw 10.8.0.200
```

- On the Kali Agent

We proceed to turn on IP forwarding along with IP masquerade on the remote Kali agent:

```bash
root@kali:~# echo 1 > /proc/sys/net/ipv4/ip_forward
root@kali:~# iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
With this complete, our remote target network is now fully accessible via layer 3 and we can use any tools we have to interact with the remote network.

And lastly, once again, a small tribute to Morbo:


<iframe width="560" height="315" src="https://www.youtube.com/embed/xoCZ07hwoZ4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>





