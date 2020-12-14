# Linux-WPA3-Personal

This document describes how to set up a WPA3-Personal Access Point using Linux, that can always have the latest security patches applied

**WPA3-Personal** is the only password-based Wi-Fi that as of 2020 is safe to use

WPA3-Personal published in 2018 is supported by Android 10 and macOS Catalina 10.15 or later

## Why is WPA3-Personal more secure?

• sae (explained below) is a safe exchange of secrets with forward secrecy

• sae only connects Access Points and clients that prove to have the secret, thereby mitigating the evil-twin problem

WPA2-Personal does not offer any of the above characteristics, it is unsafe to use at all without vpn. The intermediate fix for WPA2 is to always use WPA2-Enterprise eap-tls, everybody else is a victim. Once an attacker is in the WPA2 network which takes only seconds, crafted packets can be sent to any authenticated node exploiting every feature of their typically low-quality software

# Process

## Get a Compliant Adapter

As of July 2020 the Wi-Fi Alliance only certifies WPA3-capable hardware. To run WPA3-Personal, the hardware must support sae, <a href=https://en.wikipedia.org/wiki/Simultaneous_Authentication_of_Equals>Simultaneous Authentication of Equals IEEE 802.11-2016</a>

A safe way to determine that the hardware is capable of WPA3-Personal is to verify that the Wi-Fi Alliance has certified it for WPA3 using their <a href=https://www.wi-fi.org/product-finder>Product Finder</a>. At present, vendors claim WPA3 when they only support a certain feature like sae but cannot actually communicate with WPA3-Personal devices. Much hardware does work but is not directly certified

Wi-Fi under Linux generally is troublesome

A USB adapter known to work is NETGEAR A6210, other internal adapters may work if an AP-capable driver can be found

## Adapter Must support pmf protected management frames

<a href=https://en.wikipedia.org/wiki/IEEE_802.11w-2009>PMF (protected management frames) certification IEEE 802.11w</a>, also called mfp management frame protection

The only way to determine whether pmf is supported is to run it. hostapd will error on start when provided the following option:
ieee80211w=2

## Plug the Adapter in

If it is usb, it should be recognized by `lsusb` You can monitor how the command output changes in order to detect it

Most other adapters appear using `lspci` In edge cases use `lshw`

This will give you the vendor and product id which is a two-value hexadecimal number like 846:9053

## Find Driver

If the device has a driver, a network interface will appear with a name like wlan0. Wireless devices can be listed with `iw dev` Their names ususally begin with w double-u

<pre>
<strong>iw dev</strong>
phy#2
        Interface wlan0
                ifindex 7
                wdev 0x200000001
                addr 00:11:22:33:44:55
                type managed
                txpower 20.00 dBm
</pre>

phy2 is the iw numbering for the device. To see everything it can do, about 200 lines: `iw phy2 info`

All network interfaces are listed by `ip l`

One can also examine which driver claims a certain vendor and product:
<pre>
<strong>modprobe --showconfig | grep v0846p9053</strong>
alias usb:v0846p9053d*dc*dsc*dp*ic*isc*ip*in* mt76x2u
</pre>
Here we can see that the usb product **0846:9053** is claimed by the **mt76x2u** driver. A driver name is a short word

If no driver claims your product, search the vendor’s web site or search generally on the Internet, many drivers are maintained by volunteers on github. Success is more likely if a tutorial is found by someone else with the same product

If a network interface appears and you want to know which driver provides it, here is for the **wlp2s0b1** network interface:
<pre>
ls /sys/class/net/<strong>wlp2s0b1</strong>/device/driver/..
b43
</pre>
The wlp2s0b1 interface was provided by the **b43** driver

## Disable NetworkManager

To use hostapd, the device should not be managed by NetworkManager. You have NetworkManager if the directory /etc/NetworkManager/conf.d exists
There can only be one file with **unmanaged-devices** directive

Check for a file to append to:
<pre>
<strong>grep unmanaged /etc/NetworkManager/conf.d/*</strong>
/etc/NetworkManager/conf.d/00-unmanaged.conf:unmanaged-devices=interface-name:enp4s0;interface-name:enx*;interface-name:wlx08beac12be74
</pre>

If /etc/NetworkManager/conf.d does not eixst, move on

in this case, add to the line that already exists

otherwise create a file with extension .conf like:
<pre>
<strong>nano /etc/NetworkManager/conf.d/00-unmanaged.conf</strong>
[keyfile]
unmanaged-devices=interface-name:wlan0

<strong>systemctl restart NetworkManager.service</strong>
</pre>

Verify that NetworkManager no longer manages your device:
<pre>
<strong>nmcli d s</strong>
DEVICE           TYPE      STATE         CONNECTION 
wlan0            wifi      unmanaged     --         
</pre>

## Assign IP

Pick an unused area of 256 ip addresses, like 10.0.5.0. Create a file with .network extension, here for the **wlan0** interface:

<pre>
<strong>nano /etc/systemd/network/10-wlan0.network</strong>
[Match]
Name=<strong>wlan0</strong>
[Link]
RequiredForOnline=no
[Network]
ConfigureWithoutCarrier=true
Address=<strong>10.0.5.0</strong>/24

<strong>systemctl reload systemd-networkd</strong>
</pre>

`ip a` should now show that the interface has an ip address

## Get dhcp and dns

Install dnsmasq and hostapd: `apt install --yes dnsmasq hostapd`
If your interface is <strong>wlan0</strong> and ip <strong>10.0.5.0</strong>, create two files:
<pre>
<strong>nano /etc/hostapd/wlan0-dnsmasq</strong>
bind-interfaces
listen-address=<strong>10.5.0.1</strong>
no-hosts
dhcp-range=<strong>10.5.0.32,10.5.0.63</strong>
dhcp-option=option:router,<strong>10.5.0.1</strong>

<strong>nano /etc/systemd/system/dnsmasq@.service</strong>
[Unit]
Description=dhcp and dns for interface %i
Requires=network.target
Wants=nss-lookup.target
Before=nss-lookup.target
After=network.target
[Service]
PIDFile=/run/dnsmasq/%i.pid
Restart=on-failure
RestartSec=5
ExecStart=/usr/sbin/dnsmasq --keep-in-foreground --conf-file=/etc/hostapd/%i-dnsmasq
ExecReload=/bin/kill -HUP $MAINPID
[Install]
WantedBy=multi-user.target

<strong>systemctl start dnsmasq@wlan0</strong>
</pre>

dns and dhcp are now running

## Start hostapd

Create a file named using your interface **wlan0** and extension .conf:
also decide on your ssid and password. The password should not be known or guessable to an attacker
<pre>
<strong>nano /etc/hostapd/wlan0.conf</strong>
# directive order from: https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf
# © 2020-present Harald Rudell <harald.rudell@gmail.com>
# driver: mt76x2u WPA3-Personal 802.11g
interface=<strong>wlan0</strong>
ssid=<strong>anything</strong>
logger_stdout=63
logger_stdout_level=0
country_code=US
ieee80211d=1
hw_mode=g
channel=9
auth_algs=1
ap_isolate=1
wpa=2
wpa_key_mgmt=SAE
rsn_pairwise=CCMP
ieee80211w=2
sae_password=<strong>unguessable</strong>
sae_require_mfp=1
</pre>

To test you access point:
`hostapd -t /etc/hostapd/wlan0.conf`

To run your access point:
`systemctl start hostapd@wlan0`

Access Points under linux are implemented using hostapd: <a href=https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf>hostapd reference</a>

This allows Wi-Fi clients to access the host machine. For clients to access the Internet, ip forwarding and SNAT must be added; a template is here for wi-fi interface wlan0 and Internet interface eth0:
<pre>
iptables --insert FORWARD --in-interface wlan0 --out-interface eth0 --jump ACCEPT
iptables --insert FORWARD --in-interface wlan0 --out-interface eth0 --jump MARK --set-mark 0x123
iptables --insert FORWARD --in-interface eth0 --out-interface wlan0 --match state --state RELATED,ESTABLISHED --jump ACCEPT
iptables --append FORWARD --jump DROP
iptables --table nat --insert POSTROUTING --match mark 0x123 --jump MASQUERADE
</pre>
echo -n 1 >/proc/sys/net/ipv4/ip_forward

Note: as of Linux 5.4.0, bugs prevents speeds faster than 54 Mb/s 802.11g. NETGEAR A6210 can otherwise do 867 Mb/s on 5 GHz

Updated: 12/13/2020
