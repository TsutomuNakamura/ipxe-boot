# ipxe-boot
This is a repository to run containers `ipxe-server-core` and `ipxe-image-server`.
They enables iPXE clients to boot and install OS.



## Preparing configurations
### Set macvlan network
You should create macvlan that its segment is same as your local network.

Edit `.env` to adapt on your environment of network.
You can set them like below if your network is `172.31.0.0/16` for example.

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
```
# * Your network address is "192.168.1.0/24"
# * IP range that will be assigned to containers on the macvlan is "192.168.1.0 - 192.168.1.127"
# * IP of gateway is "192.168.1.1"
MACVLAN_IPXE_SUBNET="192.168.1.0/24"
MACVLAN_IPXE_IP_RANGE="192.168.1.0/25"
MACVLAN_IPXE_GATEWAY="192.168.1.1"
```

## Run containers
You can run containers with docker-compose.
It will run containers `ipex-server-core` and `ipxe-image-server`.

```
docker-compose up
```

Press `Ctrl + c` if your want to stop containers.

