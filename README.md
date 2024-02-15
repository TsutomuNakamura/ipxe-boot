# ipxe-boot
This is a repository to run containers `ipxe-server-core` and `ipxe-image-server`.
They enables iPXE clients to boot and install OS.

![Boot sequence of ipxe-boot](https://raw.githubusercontent.com/TsutomuNakamura/references/main/drawio/TsutomuNakamura/ipxe-boot/main/Diagrams-ipxe-boot-flow.svg "ipxe-boot sequence")

## Preparing configurations
### Set macvlan network (in .env)
You should create macvlan that its segment is same as your local network.

Edit `.env` to adapt on your environment of network.
You can set them like below if your network is `172.31.0.0/16` for example.

* .env
```
# It assumes that...
# * Your network address is "172.31.0.0/16"
# * IP range of containers on the macvlan is "172.31.2.0 - 172.31.2.127"
# * IP of gateway is "172.31.0.1"
MACVLAN_IPXE_SUBNET="172.31.0.0/16"
MACVLAN_IPXE_IP_RANGE="172.31.2.0/25"
MACVLAN_IPXE_GATEWAY="172.31.0.1"
```

As another example, if your network is `192.168.1.0/24` for example.

* .env
```
# * Your network address is "192.168.1.0/24"
# * IP range that will be assigned to containers on the macvlan is "192.168.1.0 - 192.168.1.127"
# * IP of gateway is "192.168.1.1"
MACVLAN_IPXE_SUBNET="192.168.1.0/24"
MACVLAN_IPXE_IP_RANGE="192.168.1.0/25"
MACVLAN_IPXE_GATEWAY="192.168.1.1"
```

### Set dnsmasq's options that iPXE client will use (in .env)
Set dnsmasq's oprions by using a variable `COMMAND_IPXE_SERVER_CORE`.
You should set `dhcp-range` and `dhcp-option` at least if you use dnsmasq on `ipxe-server-core` as main DHCP in your network.

* .env
```
# * Your network address is "172.31.0.0/16"
# * DHCP range of iPXE clients is "172.31.2.128 - 172.31.2.255"
# * IP of gateway is "172.31.0.1"
# * IP of DNS is "8.8.8.8"
COMMAND_IPXE_SERVER_CORE="--dhcp-range=172.31.2.128,172.31.2.255,255.255.0.0 --dhcp-option=option:router,172.31.0.1 --dhcp-option=option:dns-server,8.8.8.8"
```

### Set IP of TFTP server that provides iPXE firmware
Set the IP of TFTP server.
This IP will be used a container `ipxe-server-core` and it should be in the network `MACVLAN_IPXE_SUBNET`.

* .env
```
IP_TFTP_NEXT_SERVER=172.31.2.11
```

### Set IP of HTTP server that provides iPXE script and OS images
Set the IP of HTTP server.
This IP will be used a container `ipxe-image-server` and it should be in the network `MACVLAN_IPXE_SUBNET`.

* .env
```
IP_TFTP_NEXT_SERVER=172.31.2.11
```

## Run containers
You can run containers with docker-compose.
It will run containers `ipex-server-core` and `ipxe-image-server`.

```
docker-compose up
```

Press `Ctrl + c` if your want to stop containers.

