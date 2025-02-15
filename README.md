# docker-netbootxyz

[![Build Status](https://github.com/netbootxyz/docker-netbootxyz/workflows/build/badge.svg)](https://github.com/netbootxyz/docker-netbootxyz/actions?query=workflow%3Abuild)
[![Discord](https://img.shields.io/discord/425186187368595466)](https://discord.gg/An6PA2a)

## Overview

The netboot.xyz docker image allows you to easily set up a local instance of netboot.xyz with a single command. The container is a small helper application written in node.js. It provides a simple web interface for editing menus on the fly, retrieving the latest menu release of netboot.xyz, and enables mirroring the downloadable assets from Github to your location machine for faster booting of assets. 

It is a great tool for developing and testing custom changes to the menus. If you are looking to get started with netboot.xyz and don't want to manage iPXE menus, you should use the boot media instead of setting up a container. The container is built from Alpine Linux and contains several components:

* netboot.xyz [webapp](https://github.com/netbootxyz/webapp)
* Nginx for hosting local assets from the container
* tftp-hpa
* syslog for providing tftp activity logs

Services are managed in the container by [supervisord](http://supervisord.org/).

## Usage

The netboot.xyz docker image requires an existing DHCP server to be setup and running in order to boot from it. The image does not contain a DHCP server service. Please see the DHCP configuration setup near the end of this document for ideas on how to enable your environment to talk to the container. In most cases, you will need to specify the next-server and boot file name in the DHCP configuration.

The following snippets are examples of starting up the container. 

### docker-cli

```bash
docker run -d \
  --name=netbootxyz \
  -e MENU_VERSION=2.0.47 `# optional` \
  -p 3000:3000 `# sets webapp port` \
  -p 69:69/udp `# sets tftp port` \
  -p 8080:80 `# optional` \
  -v /local/path/to/config:/config `# optional` \
  -v /local/path/to/assets:/assets `# optional` \
  --restart unless-stopped \
  ghcr.io/netbootxyz/netbootxyz
```

#### Updating the image with docker-cli

```bash
docker pull ghcr.io/netbootxyz/netbootxyz   # pull the latest image down
docker stop netbootxyz                      # stop the existing container
docker rm netbootxyz                        # remove the image
docker run -d ...                           # previously ran start command
```

Start the container with the same parameters used above. If the same folders are used your settings will remain. If you want to start fresh, you can remove the paths and start over.

### docker-compose

```yaml
---
version: "2.1"
services:
  netbootxyz:
    image: ghcr.io/netbootxyz/netbootxyz
    container_name: netbootxyz
    environment:
      - MENU_VERSION=2.0.47 # optional
    volumes:
      - /path/to/config:/config # optional
      - /path/to/assets:/assets # optional
    ports:
      - 3000:3000
      - 69:69/udp
      - 8080:80 #optional
    restart: unless-stopped
```

#### Updating the image with docker-compose

```bash
docker-compose pull netbootxyz     # pull the latest image down
docker-compose up -d netbootxyz    # start containers in the background
```

### Accessing the container services

Once the container is started, the netboot.xyz web application can be accessed by the web configuration interface at http://localhost:3000 or via the specified port.

Downloaded web assets will be available at http://localhost:8080 or the specified port.  If you have specified the assets volume, the assets will be available at http://localhost:8080.

If you wish to start over from scratch, you can remove the local configuration folders and upon restart of the container, it will load the default configurations.

## Parameters

Container images are configured using parameters passed at runtime (such as those above). These parameters are separated by a colon and indicate `<external>:<internal>` respectively. For example, `-p 8080:80` would expose port `80` from inside the container to be accessible from the host's IP on port `8080` outside the container.

| Parameter | Function |
| :----: | --- |
| `-p 3000` | Web configuration interface. |
| `-p 69/udp` | TFTP Port. |
| `-p 80` | NGINX server for hosting assets. |
| `-e MENU_VERSION=2.0.47` | Specify a specific version of boot files you want to use from netboot.xyz (unset pulls latest) |
| `-v /config` | Storage for boot menu files and web application config |
| `-v /assets` | Storage for netboot.xyz bootable assets (live CDs and other files) |

## DHCP Configurations

This image requires the usage of a DHCP server in order to function properly. If you have an existing DHCP server, usually you will need to make some small adjustments to make your DHCP server forward requests to the netboot.xyz container. You will need to typically set your next-server and boot-file-name parameters in the DHCP configuration. This tells DHCP to forward requests to the TFTP server and then select a boot file from the TFTP server.

### Examples

These are a few configuration examples for setting up a DHCP server. The main configuration you will need to change are next-server and filename/boot-file-name. Next-server tells your client to check for a host running tftp and retrieve a boot file from there. Because the docker image is hosting a tftp server, the boot files are pulled from it and then it will attempt to load the iPXE configs directly from the host. You can then modify and adjust them to your needs. See [booting from TFTP](https://netboot.xyz/docs/booting/tftp/) for more information.
#### isc-dhcp-server

```
subnet 10.0.100.0 netmask 255.255.255.0 {
  range 10.0.100.100 10.0.100.200;
  next-server 10.0.100.10;
  option subnet-mask 255.255.255.0;
  option routers 10.0.100.1;
  option broadcast-address 10.0.100.255;
  option domain-name "mynetwork.lan";
  option domain-name-servers 1.1.1.1;
  if exists user-class and ( option user-class = "iPXE" ) {
    filename "http://boot.netboot.xyz/menu.ipxe";
  } elsif option client-architecture = encode-int ( 16, 16 ) {
    filename "http://boot.netboot.xyz/ipxe/netboot.xyz.efi";
    option vendor-class-identifier "HTTPClient";
  } elsif option arch = 00:07 {
    filename "netboot.xyz.efi";
  } elsif option arch = 00:00 {
    filename "netboot.xyz.pxe";
  } else {
    filename "netboot.xyz.kpxe";
  }          
}
```

#### TODO - Add more examples
## netboot.xyz boot file types

The following bootfile names can be set as the boot file in the DHCP configuration. They are baked into the Docker image:

| bootfile name      | description                                                 |
| -------------------|-------------------------------------------------------------|
| `netboot.xyz.kpxe` | Legacy DHCP boot image file, uses built-in iPXE NIC drivers |
| `netboot.xyz-undionly.kpxe` | Legacy DHCP boot image file, use if you have NIC issues |
| `netboot.xyz.efi` | UEFI boot image file, uses built-in UEFI NIC drivers |
| `netboot.xyz-snp.efi` | UEFI w/ Simple Network Protocol, attempts to boot all net devices |
| `netboot.xyz-snponly.efi` | UEFI w/ Simple Network Protocol, only boots from device chained from |
| `netboot.xyz-arm64.efi` | DHCP EFI boot image file, uses built-in iPXE NIC drivers |
| `netboot.xyz-arm64-snp.efi` | UEFI w/ Simple Network Protocol, attempts to boot all net devices |
| `netboot.xyz-arm64-snponly.efi` | UEFI w/ Simple Network Protocol, only boots from device chained from |
| `netboot.xyz-rpi4-snp.efi` | UEFI for Raspberry Pi 4, attempts to boot all net devices |
