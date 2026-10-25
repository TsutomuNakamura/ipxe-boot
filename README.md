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
`--dhcp-option=option:dns-server` is required in this case. It is the DNS server that clients will use,
because dnsmasq in `ipxe-server-core` does not work as a DNS server.
The DHCP range must not overlap `MACVLAN_IPXE_IP_RANGE`.

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
IP_HTTP_NEXT_SERVER=172.31.2.12
```

### Customize OS installations (optional)
`ipxe-image-server` uses its built-in `config.yml`, `boot.ipxe.j2` and `templates/` by default.
You can override them by putting your own files in `etc/ipxe-image-server/`.
It is mounted on `/etc/ipxe-image-server` of the container `ipxe-image-server`.

```
etc/ipxe-image-server/
├── config.yml      # Menus, autoinstall configs and OS images
├── boot.ipxe.j2    # Template of the iPXE script
└── templates/      # Templates of autoinstall configs (cloud-init user-data)
```

* Each file is optional. Only the files you put there will be overridden.
* `config.yml` replaces the built-in one entirely, so it must contain all of `menus`, `autoinstalls` and `images`.
* If you put `templates/`, it replaces all of the built-in templates.
* Files in this directory except `README.md` are ignored by git, because they may contain passwords or SSH keys.

You can see the built-in files in [TsutomuNakamura/ipxe-image-server](https://github.com/TsutomuNakamura/ipxe-image-server) as examples.
Restart the container `ipxe-image-server` after you changed them.

## Run containers
You can run containers with Docker Compose.
It will run containers `ipxe-server-core` and `ipxe-image-server`.

```
docker compose up
```

Press `Ctrl + c` if your want to stop containers.

## Requirements of clients
* RAM: about 16 GiB for Ubuntu Server. The installer downloads the whole ISO into memory.
* UEFI Secure Boot must be disabled, because `ipxe.efi` is not signed.

## Test iPXE boot with QEMU
See [TESTING.md](TESTING.md) for a full test with a KVM VM, from booting iPXE to logging in to the installed OS.

```bash
mkdir -p /var/kvm/distros/test-ipxe-boot
# virt-install without specifiing mac address
virt-install \
  --name test-ipxe-boot \
  --boot uefi \
  --pxe \
  --ram 16384 \
  --os-variant=ubuntu24.04 \
  --disk /var/kvm/distros/test-ipxe-boot/disk.img,size=16,bus=virtio,format=qcow2 \
  --network bridge=br0

# virt-install with specifiing mac address
virt-install \
  --name test-ipxe-boot \
  --boot uefi \
  --pxe \
  --ram 16384 \
  --os-variant=ubuntu24.04 \
  --disk /var/kvm/distros/test-ipxe-boot/disk.img,size=16,bus=virtio,format=qcow2 \
  --network bridge=br0,mac=52:54:ff:00:00:01
```

