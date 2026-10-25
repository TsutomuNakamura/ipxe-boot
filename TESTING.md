# Testing ipxe-boot with a KVM VM

This document describes how to test the whole stack: run `ipxe-server-core` and `ipxe-image-server` with Docker,
boot a KVM VM from the network, and check that it installs Ubuntu automatically.

It assumes that:

* The host has a bridge `br0` on the network `172.31.0.0/16`, like `.env`.
* The network already has a DHCP server (e.g. the router `172.31.0.1`), so `ipxe-server-core` runs in DHCP proxy mode.
  If it does not, refer to [If your network has no DHCP server](#if-your-network-has-no-dhcp-server).
* libvirt (`virt-install`, `virsh`) is installed, and you can run it with `sudo`.
* About 20 GiB of free disk space for the VM and the images.

## 1. Prepare the configuration
### `.env.test`: DHCP proxy mode
Create a copy of `.env` that runs dnsmasq in DHCP proxy mode, and ignore it locally.
Proxy mode only adds boot information to the DHCP of your network, so it does not give addresses to other hosts.

```bash
sed 's/^COMMAND_IPXE_SERVER_CORE=.*/COMMAND_IPXE_SERVER_CORE=""/' .env > .env.test
grep -qx '.env.test' .git/info/exclude || echo '.env.test' >> .git/info/exclude
grep -n '^COMMAND_IPXE_SERVER_CORE\|^TAG_' .env.test
```

Do not run `ipxe-server-core` as a DHCP server in a network that already has one,
because two DHCP servers would give addresses to the same hosts.

### If your network has no DHCP server
DHCP proxy mode does not give IP addresses to clients. It only adds boot information to the replies of another DHCP server,
so clients cannot boot in a network without a DHCP server.
Run `ipxe-server-core` as the DHCP server instead, with `.env` as it is:

* Skip creating `.env.test`, and use `docker compose` without `--env-file .env.test` in the following steps.
* Check that `COMMAND_IPXE_SERVER_CORE` in `.env` fits your network.

```bash
grep -n '^COMMAND_IPXE_SERVER_CORE\|^MACVLAN_IPXE_IP_RANGE' .env
# COMMAND_IPXE_SERVER_CORE="--dhcp-range=172.31.2.128,172.31.2.255,255.255.0.0 --dhcp-option=option:router,172.31.0.1 --dhcp-option=option:dns-server,8.8.8.8"
```

| Option | What to set |
|--------|-------------|
| `--dhcp-range` | Addresses that no other host uses, outside of `MACVLAN_IPXE_IP_RANGE` (`172.31.2.0/25`) |
| `--dhcp-option=option:router` | The gateway of the network. Without access to the internet, the installer cannot reach package mirrors, but installs from the ISO |
| `--dhcp-option=option:dns-server` | Required, because dnsmasq of `ipxe-server-core` does not work as a DNS server. `ipxe-server-core` warns if it is missing |

### `etc/ipxe-image-server/config.yml`: SSH key and DHCP for the test VM
Copy the built-in `config.yml` of `ipxe-image-server`. Files in `etc/ipxe-image-server/` are ignored by git.

```bash
docker run --rm --entrypoint cat tsutomu/ipxe-image-server:v0.5.0 /config.yml > etc/ipxe-image-server/config.yml
```

Then replace its `autoinstalls` with the following, and set your public key (e.g. `cat ~/.ssh/id_ed25519.pub`).
The test VM gets an address by DHCP, so that it does not use a static address that may be used by another host.

```yaml
autoinstalls:
  - id: common
    template: common.yml.j2
    args:
      authorized-keys:
        - "ssh-ed25519 AAAA... you@example.com"

  - id: "52:54:ff:00:00:01"
    template: autoinstall_by_mac.yml.j2
    args:
      hostname: ipxe-test
      authorized-keys:
        - "ssh-ed25519 AAAA... you@example.com"
      network:
        version: 2
        ethernets:
          lan:
            match: {name: "en*"}
            dhcp4: true
```

Check the result.

```bash
docker compose --env-file .env.test config | grep -E 'image:|ip_range|ipv4_address|command'
```

## 2. Start the iPXE environment
```bash
docker compose --env-file .env.test up -d
docker compose --env-file .env.test ps                    # both containers should be "running"
docker logs -f ipxe-image-server                           # wait for "Starting nginx", then Ctrl+C
docker logs ipxe-server-core | grep -E 'dnsmasq|WARNING'   # "DNS disabled" and "proxy on subnet 172.31.2.11"
```

The host cannot reach containers on a macvlan network directly.
To check HTTP, use a temporary container on the same network with a free address (e.g. `172.31.2.13`).

```bash
ping -c1 -W1 172.31.2.13 || echo "172.31.2.13 is free"
docker run --rm --net macvlan_ipxe --ip 172.31.2.13 alpine:3.24 sh -c '
  for p in os/config/boot.ipxe os/images/ubuntu-24.04-live-server-amd64/casper/vmlinuz os/autoinstall/52:54:ff:00:00:01/user-data; do
    printf "%s: " $p; wget -q -S -O /dev/null http://172.31.2.12/$p 2>&1 | grep -m1 HTTP; done'
```

## 3. Create the VM that boots from the network
```bash
sudo mkdir -p /var/kvm/distros/test-ipxe-boot
sudo virt-install --connect qemu:///system \
  --name test-ipxe-boot \
  --boot uefi,firmware.feature0.name=secure-boot,firmware.feature0.enabled=no \
  --pxe --ram 16384 --vcpus 2 --os-variant=ubuntu24.04 \
  --disk /var/kvm/distros/test-ipxe-boot/disk.img,size=16,bus=virtio,format=qcow2 \
  --network bridge=br0,mac=52:54:ff:00:00:01 \
  --graphics vnc,listen=127.0.0.1 --noautoconsole
```

* The MAC `52:54:ff:00:00:01` is registered in `autoinstalls`, so the menu "with autoinstaller (MAC: ...)" works.
* Secure Boot must be disabled, because `ipxe.efi` is not signed.
* The installer needs about 16 GiB of RAM, because it downloads the whole ISO into memory.

## 4. Watch the boot and choose an installation
Open the screen of the VM (or use `virt-manager`).

```bash
sudo virt-viewer --connect qemu:///system test-ipxe-boot
```

Choose "Install Ubuntu 24.04 LTS with autoinstaller (MAC: 52:54:ff:00:00:01)".
The menu should show the MAC of the VM. Watch the logs in other terminals.

```bash
docker logs -f ipxe-server-core     # PXE(eth0) 52:54:ff:00:00:01 proxy, sent /pxeboot/firmware/ipxe.efi
docker logs -f ipxe-image-server    # vmlinuz, initrd, the .iso, then meta-data, user-data and vendor-data, all 200
```

The installation runs without any input and takes about 10 minutes.

## 5. Boot the installed system and log in
`virt-install` with `--noautoconsole` leaves the VM shut off after the installation. Start it.

```bash
sudo virsh domstate test-ipxe-boot          # "shut off" when the installation finished
sudo virsh start test-ipxe-boot
```

The VM gets an address from the DHCP server of your network.
Find it in the DHCP leases of your router, or by the MAC of the VM after pinging the range of your DHCP server once.

```bash
# Replace 192.0.2 with the first three numbers of the DHCP range of your network
for h in $(seq 1 254); do ping -c1 -W1 192.0.2.$h >/dev/null 2>&1 & done; wait
ip neigh show dev br0 | grep -i 52:54:ff:00:00:01

ssh -i ~/.ssh/id_ed25519 me@<address of the VM>
```

If `ipxe-server-core` runs as the DHCP server, it logs the addresses that it gives.
iPXE, the installer and the installed system may get different addresses, so use the last one.

```bash
docker logs ipxe-server-core | grep 'DHCPACK.*52:54:ff:00:00:01'
# dnsmasq-dhcp[9]: DHCPACK(eth0) 172.31.2.150 52:54:ff:00:00:01 ipxe-test
```

## 6. Check the installed system
```bash
ssh -i ~/.ssh/id_ed25519 me@<address of the VM> \
  'hostname; . /etc/os-release; echo $PRETTY_NAME; ip -br -4 addr; ip route show default; getent hosts archive.ubuntu.com'
# ipxe-test, Ubuntu 24.04, an address from DHCP, a default route and working name resolution

ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password me@<address of the VM>
# "Permission denied (publickey)", because password login over SSH is disabled
```

On the console of the VM, you can log in as `me` with the password `p@ssw0rd`. `sudo` asks for this password.

## 7. Other cases
| Case | How | Expected |
|------|-----|----------|
| BIOS instead of UEFI | Remove `--boot uefi,...` in step 3 | dnsmasq sends `ipxe.pxe` instead of `ipxe.efi`, and the same menu is shown |
| A MAC that is not registered | Use e.g. `mac=52:54:ff:00:00:99` in step 3 | The menu shows the MAC. `/os/autoinstall/52:54:ff:00:00:99/...` returns 404 and the installer becomes interactive |
| The common autoinstaller | Choose "... with common autoinstaller" | `/os/autoinstall/common/` is used, and the hostname is `default-server` |
| Advanced options of the menu | "Configure settings", "Drop to iPXE shell" then `exit`, or Esc | The settings screen, the menu without the timeout, or the shell |
| An error in the configuration | Set a template that does not exist in `config.yml`, then `docker compose --env-file .env.test up -d` | `ipxe-image-server` exits and `docker logs ipxe-image-server` lists the errors |
| Ubuntu 22.04 | Choose an Ubuntu 22.04 menu | The same flow |

Remove the VM before you try another case (see the next section), or use another `--name` and disk.

## 8. Clean up
```bash
docker compose --env-file .env.test down
sudo virsh destroy test-ipxe-boot
sudo virsh undefine test-ipxe-boot --nvram
sudo rm -rf /var/kvm/distros/test-ipxe-boot

# Optional: remove the files for the test
rm .env.test etc/ipxe-image-server/config.yml
sed -i '/^\.env\.test$/d' .git/info/exclude
```

Older versions of `ipxe-image-server` extracted the whole `casper/` of each image.
If `ipxe-image-server` warns about them, refer to "Removing files extracted by older versions" in the README.md of
[TsutomuNakamura/ipxe-image-server](https://github.com/TsutomuNakamura/ipxe-image-server).
