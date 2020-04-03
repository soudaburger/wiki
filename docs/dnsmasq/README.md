# Configure DNSmasq to support TFTP and PXE clients
<!-- TOC depthFrom:3 -->
- [Configure DNSmasq to support TFTP and PXE clients](#configure-dnsmasq-to-support-tftp-and-pxe-clients)
- [Assumptions](#assumptions)
- [Prep PXE Host](#prep-pxe-host)
  - [Initial System Prep](#initial-system-prep)
  - [Create OS-specific folders to support clients](#create-os-specific-folders-to-support-clients)
  - [Set up kickstart files to automate provisioning](#set-up-kickstart-files-to-automate-provisioning)
  - [Set up and configure dnsmasq](#set-up-and-configure-dnsmasq)
  - [Install and configure Nginx to support kickstart clients](#install-and-configure-nginx-to-support-kickstart-clients)


---
# Assumptions
    * This has ONLY been tested on Ubuntu 18.04 systems.
  
    * It is assumed that /tftp is the root of the TFTP directory being served to PXE clients.

    * All commands are run on the PXE server unless otherwise specified.
---
# Prep PXE Host

## Initial System Prep
```bash
apt update
apt install -y pxelinux syslinux

mkdir -p /tftp/pxelinux.cfg

cp /usr/lib/PXELINUX/pxelinux.0 /tftp/pxelinux.0

cp /usr/lib/syslinux/modules/bios/{ldlinux.c32, libcom32.c32, libutil.c32, vesamenu.c32} /tftp/
```

## Create OS-specific folders to support clients
> NOTE: KICKSTART_IP should be changed according to your environment.

```bash
KICKSTART_IP=1.2.3.4

mkdir /opt/isos
mkdir /tftp/ubuntu1804
wget -c -O /opt/isos/ubuntu1804.iso "http://cdimage.ubuntu.com/releases/18.04/release/ubuntu-18.04.4-server-amd64.iso"
mount -o loop /opt/isos/ubuntu1804.iso /tftp/ubuntu1804

cat << EOF >> /tftp/pxelinux.cfg/default
label ubuntu1804
  menu label ^Install Ubuntu 18.04 LTS Server
  kernel ubuntu1804/install/netboot/ubuntu-installer/amd64/linux
  append initrd=ubuntu1804/install/netboot/ubuntu-installer/amd64/initrd.gz auto=true vga=768 hostname=ubuntu1804 url=http://${KICKSTART_IP}/kickstart/ubuntu1804.cfg
EOF
```

## Set up kickstart files to automate provisioning
> NOTE: This file was generated from an ansible template.  Replace jinja variables as necessary.
```bash
mkdir /tftp/kickstart

cat << EOF > /tftp/kickstart/ubuntu1804.cfg
# network settings
d-i netcfg/choose_interface select auto

# networking with DHCP:
d-i netcfg/disable_autoconfig boolean false

# regional setting
d-i debian-installer/language string en_US:en
d-i debian-installer/country string US
d-i debian-installer/locale string en_US
d-i debian-installer/splash boolean false
d-i localechooser/supported-locales multiselect en_US.UTF-8
d-i pkgsel/install-language-support boolean true

# keyboard selection
d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i keyboard-configuration/modelcode string pc105
d-i keyboard-configuration/layoutcode string us
d-i keyboard-configuration/variantcode string intl
d-i keyboard-configuration/xkb-keymap select us(intl)
d-i debconf/language string en_US:en

# mirror settings
d-i mirror/country string manual
d-i mirror/http/hostname string us.archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/http/proxy string

# clock and timezone settings
d-i time/zone string America/New_York
d-i clock-setup/utc boolean false
d-i clock-setup/ntp boolean true

# user account setup
d-i passwd/root-login boolean false
# printf "password" | mkpasswd -s -m sha-512
d-i passwd/root-password-crypted password {{ ubuntu_root_password_hash }}
d-i passwd/make-user boolean true
d-i passwd/user-fullname string {{ ubuntu_user }}
d-i passwd/username string {{ ubuntu_user }}
# printf "password" | mkpasswd -s -m sha-512
d-i passwd/user-password-crypted password {{ ubuntu_user_password_hash }}
d-i passwd/user-uid string
d-i user-setup/allow-password-weak boolean false
#d-i passwd/user-default-groups string adm cdrom dialout lpadmin plugdev sambashare
d-i user-setup/encrypt-home boolean false

# configure apt, and install sshd
d-i apt-setup/restricted boolean true
d-i apt-setup/universe boolean true
d-i apt-setup/backports boolean true
d-i apt-setup/services-select multiselect security
d-i apt-setup/security_host string security.ubuntu.com
d-i apt-setup/security_path string /ubuntu
tasksel tasksel/first multiselect openssh-server
d-i pkgsel/upgrade select safe-upgrade
d-i pkgsel/update-policy select none
d-i pkgsel/updatedb boolean true

# disk partitioning
# TIP: you can comment all of this out and do only this step manually.
# More complex recipes are also possible.
d-i partman-auto/choose_recipe select atomic
d-i partman/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm_nooverwrite boolean true
d-i partman/confirm boolean true
d-i partman-auto/purge_lvm_from_device boolean true
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-auto-lvm/no_boot boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-md/confirm boolean true
d-i partman-md/confirm_nooverwrite boolean true
d-i partman-auto/method string regular
d-i partman-auto-lvm/guided_size string max
d-i partman-partitioning/confirm_write_new_label boolean true

# grub boot loader
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true

# finish installation
d-i finish-install/reboot_in_progress note
d-i finish-install/keep-consoles boolean false
d-i cdrom-detect/eject boolean true
d-i debian-installer/exit/halt boolean false
d-i debian-installer/exit/poweroff boolean false
EOF
```

## Set up and configure dnsmasq
```bash
apt update
apt install -y dnsmasq

cat << EOF > /etc/dnsmasq.conf
enable-tftp
# interface=kickstart_interface
domain=changeme
 
tftp-root=/tftp
tftp-no-blocksize

log-facility=/var/log/dnsmasq/dnsmasq.log
log-async
log-queries
EOF

service dnsmasq restart
``` 

## Install and configure Nginx to support kickstart clients
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