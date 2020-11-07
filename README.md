# WireGuard® on Docker with OVPN
This document describes the process of creating a [WireGuard®](https://www.wireguard.com/) interface to the excellent VPN service [OVPN](https://www.ovpn.com/) for docker containers without routing all traffic for the server through the wg interface.
The guide is tested for CentOS 7 but should work for CentOS 8 and other similar RHEL distributions as well.

This is a work in progress, for improvements and corrections, please open a pull request.

Some good starting resources: [RHEL 7](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bonding_using_the_command_line_interface
) and [RHEL 8](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-network-bonding_configuring-and-managing-networking)

## Install WireGuard®
Install according to the instructions on [CentOS 7](https://www.wireguard.com/install/#centos-7-module-module-dkms-tools) or [CentOS 8](https://www.wireguard.com/install/#centos-8-module-kmod-module-dkms-tools).
At the time of writing:
```
$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
$ sudo yum install kmod-wireguard wireguard-tools
```

## Create docker network
Create the docker network that you want to tunnel.
```
docker network create docker-wg0 --subnet 10.50.0.0/16 -o "com.docker.network.driver.mtu"="1420"
```

## Create the WireGuard® configuration file
Download the WireGuard® configuration files from [here](https://www.ovpn.com/account/wireguard/configurations)
It's important to note the IPv4 Address `172.<...>/32` for later use.

```
[Interface]
PrivateKey = <...>
Address = 172.<...>/32, fd00:0000:1337:cafe:<...>/128
DNS = 46.227.67.134, 192.165.9.158

[Peer]
PublicKey = <...>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = vpn<...>.prd.<location>.ovpn.com:9929
```
Since we will set the IP address on the interface we can't have the `Address` and `DNS` setting in the .conf-file.
Add `PersistentKeepalive = 25` to keep the connection alive.

The final configuration should be placed in `/etc/wireguard/wg0.conf`:
```
[Interface]
PrivateKey = <...>

[Peer]
PublicKey = <...>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = vpn<...>.prd.<...>.ovpn.com:9929
PersistentKeepalive = 25
```

## Create network configurations
Add the configuration file for the WireGuard® interface `wg0` as well as the rules and routes.
The `IPADDR=172.<...>` must be the same as the IP that was removed from the .conf previously.

`/etc/sysconfig/network-scripts/ifcfg-wg0`
```
DEVICE=wg0
NAME=wg0
TYPE=wireguard
IPADDR=172.<...>
NETMASK=255.255.255.0
ONBOOT=yes
ZONE=public
NM_CONTROLLED=no
```

`/etc/sysconfig/network-scripts/route-wg0`
```
default via 172.<...> metric 2 table wg
blackhole default metric 3 table wg
```

`<local ip>` can for example be 192.168.0.0 if thats your local ip range.

`/etc/sysconfig/network-scripts/rule-wg0`
```
from 10.50.0.0/16 table wg
from 10.50.0.0/16 to <local ip>/16 table main
from 10.50.0.0/16 to 10.50.0.0/16 table main
```

`/etc/iproute2/rt_tables`
```
200     wg
```

# Create `wg0` interface on boot
To create the interface on boot we need a systemd-service.

`/etc/systemd/system/wg.service`
```
[Unit]
Description = Creating wg and bond0 interface
After = network.target

[Service]
ExecStart = /etc/systemd/system/wg.sh

[Install]
WantedBy = multi-user.target
```
Remember to `chmod +x /etc/systemd/system/wg.sh` after creating the file.

`/etc/systemd/system/wg.sh`
```
#! /bin/bash
ip link del dev wg0

ip link add dev wg0 type wireguard
wg setconf wg0 /etc/wireguard/wg0.conf

systemctl restart network

sysctl -w net.ipv4.conf.all.rp_filter=2
sysctl -w net.ipv4.ip_forward=1

ifup wg0
```

# Reboot server
After a reboot it should have conected to OVPN, `sudo wg`

```
interface: wg0
  public key: <...>
  private key: (hidden)
  listening port: <...>

peer: 
  endpoint: 
  allowed ips: 0.0.0.0/0, ::/0
  latest handshake: <...> seconds ago
  transfer: <...> received, <...> sent
  persistent keepalive: every 25 seconds
```

## Testing the interface
Check that the connection to OVPN works, you may need to install python for formatting to work.

```
sudo echo -n "my ip: " && curl -m 3 -w "\n" ifconfig.me/ip && echo -n "wg ip: " && \
sudo docker run -ti --rm --net=docker-wg0 appropriate/curl -m 3 -w "\n" https://www.ovpn.com/v2/api/client/ptr | python -m json.tool
```
or if you are using a different VPN provider:
```
sudo echo -n "my ip: " && curl -m 3 -w "\n" ifconfig.me/ip && echo -n "wg ip: " \
&& sudo docker run -ti --rm --net=docker-wg0 appropriate/curl -m 3 -w "\n" ifconfig.me/ip | python -m json.tool
```

for other testing use `network-multitool` or `ubuntu` docker images:

`sudo docker run -it --rm --net=docker-wg0 praqma/network-multitool /bin/bash`

`sudo docker run -it --rm --net=docker-wg0 ubuntu /bin/bash`

## Using the interface with docker compose

```
networks:
  docker-wg0:
    external:
      name: docker-wg0

services:
  ubuntu:
    image: ubuntu
    container_name: ubuntu
    networks:
      - docker-wg0
```

WireGuard® is a registered trademark of Jason A. Donenfeld.
