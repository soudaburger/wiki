# PXE Boot ESXI 6.7 Targets
<!-- TOC depthFrom:3 -->
- [PXE Boot ESXI 6.7 Targets](#pxe-boot-esxi-67-targets)
- [Assumptions](#assumptions)
- [Prep PXE Host](#prep-pxe-host)
  - [Initial System Prep](#initial-system-prep)
  - [Setup Nginx](#setup-nginx)
  - [Fetch Files](#fetch-files)
    - [Prep Syslinux](#prep-syslinux)
    - [Prep ESXI 6.7 ISO](#prep-esxi-67-iso)
    - [Create ESXI 6.7 Boot Config](#create-esxi-67-boot-config)
    - [Create ESXI 6.7 Kickstart Config](#create-esxi-67-kickstart-config)
  - [Update DHCP Options](#update-dhcp-options)
  - [Validate PXE Configuration](#validate-pxe-configuration)
  - [Troubleshooting](#troubleshooting)


---
# Assumptions
    * This has ONLY been tested on Ubuntu 18.04 systems.
  
    * It is assumed that /tftp is the root of the TFTP directory being served to PXE clients.

    * This walkthrough assumes that you have already successfully configured a PXE host.

    * This walkthrough assumes that you are successfully able to download an ESXI 6.7 ISO.

    * All commands are run on the PXE server unless otherwise specified.

    * ESXI 6.7 ISO exists on PXE host at /opt/isos/esxi67.iso
---
# Prep PXE Host

## Initial System Prep
```bash
apt update
```

## Setup Nginx
```bash
apt install -y nginx
rm -rf /etc/nginx/fastcgi*
rm -rf /etc/nginx/koi*
rm -rf /etc/nginx/*params
rm -rf /etc/nginx/win-utf
rm -rf /etc/nginx/*available
rm -rf /etc/nginx/*enabled
rm -rf /etc/nginx/snippets
cat << EOF > /etc/nginx/conf.d/pxe.conf
server {
  listen *:80;
  access_log /var/log/nginx/pxe.access.log;
  error_log /var/log/nginx/pxe.error.log;
  gzip on;
  location / {
      autoindex on;
      root /tftp;
  }
}
EOF
service nginx restart
```

## Fetch Files

### Prep Syslinux
> ESXI uses a super old syslinux version.  Newer versions that come with modern operating systems are incompatible.
```bash
cd /tmp
wget https://cdn.kernel.org/pub/linux/utils/boot/syslinux/3.xx/syslinux-3.86.tar.gz
tar xf syslinux-3.86.tar.gz
cp syslinux-3.86/gpxe/gpxelinux.0 /tftp/
```

### Prep ESXI 6.7 ISO
> Fetching the ESXI 6.7 ISO is beyond the scope of this documentation.  
```bash
mkdir /tftp/esxi67
mount -o loop /opt/isos/esxi67.iso /tftp/esxi67
```

### Create ESXI 6.7 Boot Config
> Replace 1.2.3.4 with the target IP of your PXE server.

> This is modifying the provided boot.cfg so we can boot over HTTP.
```bash
cp /tftp/esxi67/boot.cfg /tmp
PXE_IP="1.2.3.4"
cat /tmp/boot.cfg | sed -e 's/=\//=/g' -e 's/ \// http:\/\/${PXE_IP}\/esxi67\//g' -e 's/^kernel=/kernel=http:\/\/${PXE_IP}\/esxi67\//g' | tee -a /tftp/esxi67.boot.cfg
```

### Create ESXI 6.7 Kickstart Config
```bash
cat << EOF > /tftp/esxi67.kickstart.cfg
#
# Sample scripted installation file
#

# Accept the VMware End User License Agreement
vmaccepteula

# Set the root password for the DCUI and Tech Support Mode
rootpw password

# Install on the first local disk available on machine
clearpart --alldrives  --overwritevmfs
install --ignoressd --firstdisk=usb --overwritevmfs --novmfsondisk

# Set the network to DHCP on the first network adapter
network --bootproto=dhcp --device=vmnic0

keyboard 'US Default'

reboot

# A sample post-install script
%post --interpreter=python --ignorefailure=true
import time
stampFile = open('/finished.stamp', mode='w')
stampFile.write( time.asctime() )
EOF
```

## Update DHCP Options

> Your DHCP server should send clients to your PXE host with the filename of gpxelinux.0 for all ESXI targets.

## Validate PXE Configuration

> PXE Boot target ESXI hosts and validate the build process.

## Troubleshooting

> See /var/log/dnsmasq.log for TFTP/DHCP troubleshooting as appropriate.

> See /var/log/nginx/error.log for HTTP troubleshooting.
