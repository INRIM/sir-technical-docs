# IKEv2 remote access VPN with Ubuntu Linux
If you don't have a Cisco device, or you want to build a backup VPN, you can create a VPN similar to [Cisco FlexVPN](vpn.md) 
using a Linux server with [strongSwan](https://www.strongswan.org/).

This guide assumes a working RADIUS server to perform EAP authentication of remote access clients. Refer to the [FlexVPN guide](vpn.md)
for an example using [OpenLDAP](https://www.openldap.org/) and [FreeRADIUS](https://freeradius.org/).

## Architecture
This guide will be based on:
- A Linux-based server (we used [Ubuntu server](https://ubuntu.com/server) 20.04).
- [strongSwan](https://www.strongswan.org/) IKEv2 server.
- [FRRouting](https://frrouting.org/) OSPF server.

## Network setup
The server will have two Ethernet interfaces:

- The main network interface, with a public IPv4 address and a global unicast IPv6 address, both static.
- A point-to-point interface to the main router.

The second interface is optional. However, it simplifies the network flow, avoiding sending ICMP
redirect packets.

The VPN will be [route-based](https://wiki.strongswan.org/projects/strongswan/wiki/RouteBasedVPN) using a
virtual XFRM interface. This allows the use of OSPF to propagate the network of virtual IPs assigned
to VPN clients.

### IP addresses
For this example, we will assume the following IP addresses:
| Interface | IPv4 address   | IPv6 address     |
| --------- | ------------   | -------------    |
|    eth0   |  192.0.2.2/24  |  2001:db8::2/64  |
|    eth1   |  172.16.0.2/30 |                  |
|    xfrm0  |  10.0.0.1/24   |  2001:db8:1:1/64 |

### systemd-networkd configuration
Any method can be used to apply the network settings. For this guide, I will be using
[systemd-networkd](https://wiki.archlinux.org/title/systemd-networkd), which is simple and easy to set up.

To do so, create the following files:

`/etc/systemd/network/20-public.network`:
```
[Match]
Name=eth0

[Network]
Address=192.0.2.2/24
Address=2001:db8::2/64
Gateway=192.0.2.1
Gateway=2001:db8::1
DNS=8.8.8.8
DNS=8.8.4.4
DNS=2001:4860:4860::8888
DNS=2001:4860:4860::8844
Xfrm=xfrm0
```
`/etc/systemd/network/21-ptp.network`:
```
[Match]
Name=eth1

[Network]
Address=172.16.0.2/30
```
`/etc/systemd/network/22-vpn-xfrm.netdev`:
```
[NetDev]
Name=xfrm0
Kind=xfrm

[Xfrm]
Independent=no
InterfaceId=48
```
`/etc/systemd/network/22-vpn-xfrm.network`:
```
[Match]
Name=xfrm0

[Network]
Address=10.0.0.1/24
Address=2001:db8:1::1/64
```
Then, reload the configuration with
```bash
systemctl restart systemd-networkd
```

## strongSwan configuration
### Installation
[strongSwan](https://www.strongswan.org/) has multiple interfaces. For this guide, I will be using
[charon-systemd](https://wiki.strongswan.org/projects/strongswan/wiki/Charon-systemd), based on the modern
[swanctl](https://wiki.strongswan.org/projects/strongswan/wiki/Swanctl) configuration file backend.

For Ubuntu 20.04 LTS, install the following packages:
```bash
sudo apt install charon-systemd libcharon-extra-plugins
```
`libcharon-extra-plugins` is needed for the [eap-radius](https://wiki.strongswan.org/projects/strongswan/wiki/EapRadius)
user authentication plugin.

### X.509 certificates
User EAP authentication requires a valid X.509 certificate for server authentication.
You can generate your own CA and certificate, or get it from a well-known CA. 

If you want to build your own CA, you can refer to https://wiki.strongswan.org/projects/strongswan/wiki/SimpleCA.
Please note that you'll need to load this CA into every client.

For this guide, I got the certificate from a well-known CA. Note that, if your RADIUS
server supports tunneled authentication methods (EAP-PEAP and EAP-TTLS), some clients will complain 
if this certificate is different from the RADIUS server's certificate.

Once got the certificate, copy the private key in `/etc/swanctl/rsa`, the X.509 certificate
in `/etc/swanctl/x509` and the CA certificate in `/etc/swanctl/x509ca`. 
I assume that the X509 certificate is called `mycert.pem`, and it is issued to `vpn.example.com`, which
points to the public IPv4 and IPv6 addresses of the server.

Reload and verify that they are successfully loaded:
```bash
systemctl restart strongswan
swanctl -x
```

### Tunnel configuration
Create a configuration file `/etc/swanctl/conf.d/vpn.conf`:
```
connections {
	my-vpn {
		version = 2
		proposals = aes256gcm16-prfsha384-ecp384,aes256-sha384-sha256-ecp256,default
		rekey_time = 24h
		pools = vpn-pool-ipv4, vpn-pool-ipv6
		fragmentation = yes
		dpd_delay = 30s
		send_cert = always
		
		local-1 {
			certs = mycert.pem
			id = vpn.example.com
		}
	
		remote-1 {
			auth = eap-radius
			eap_id = %any
		}
	
		children {
			my-vpn {
				local_ts = 0.0.0.0/0, ::/0
				rekey_time = 0s
				dpd_action = restart
				esp_proposals = aes128gcm16,aes256gcm16,aes256-sha256,aes128-sha1,default
			        if_id_in = 48
			        if_id_out = 48
			}
		}
	}
}

pools {
	vpn-pool-ipv4 {
		addrs = 10.0.0.2-10.0.0.254
	}	
	vpn-pool-ipv6 {
		addrs = 2001:db8:1:1::/68
	}

}
```

Reload and verify that this configuration is successfully loaded:
```bash
systemctl restart strongswan
swanctl -L
```

## OSPF configuration
### FRRouting
Install the latest version of [FRRouting](https://frrouting.org/) from https://deb.frrouting.org/. 

Then, open `vtysh`, enter the configuration mode with
```
# conf t
```
and add the following configuration:
```
ip forwarding
ipv6 forwarding
!
interface eth1
 ip ospf area 0
!
router ospf
 ospf router-id 172.16.0.2
 redistribute connected route-map ospf2-redistribute-rm
 passive-interface default
 no passive-interface eth1
!
router ospf6
 ospf6 router-id 172.16.0.2
 redistribute connected route-map ospf3-redistribute-rm
 interface eth1 area 0
!
ip prefix-list ospf2-redistribute-pl seq 10 permit 10.0.0.0/24
ip prefix-list ospf2-redistribute-pl seq 20 deny any
!
ipv6 prefix-list ospf3-redistribute-pl seq 10 permit 2001:db8:1::/64
ipv6 prefix-list ospf3-redistribute-pl seq 20 deny any
!
route-map ospf3-redistribute-rm permit 10
 match ipv6 address prefix-list ospf3-redistribute-pl
!
route-map ospf3-redistribute-rm deny 20
!
route-map ospf2-redistribute-rm permit 10
 match ip address prefix-list ospf2-redistribute-pl
!
route-map ospf2-redistribute-rm deny 20
```
Exit from configuration mode, save and exit
```
# exit
# write memory
# exit
```
This configuration activates OSPFv2 (for IPv4) and OSPFv3 (for IPv6) to redistribute the prefix assigned to the VPN users.
Using a route-map we filter the prefixes, making sure that we are redistributing only the VPN prefix. Finally,
this configuration also enables IPv4 and IPv6 forwarding.

### Router configuration
This configuration strongly depends on the router, and the current OSPF (v2 and v3) configuration of your network.

#### License
This guide is released under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png)
