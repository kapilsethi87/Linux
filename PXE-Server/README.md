# PXE-Server

Step: 1
**To install and Configure pxe server on centos 7.x we need the following packages “dhcp, tftp-server, ftp server(vsftpd), xinted”.**
```
yum install httpd xinetd syslinux tftp-server dhcp* -y
```

Step: 2

```
vi /etc/dnsmasq.conf

interface=ens192,lo
#bind-interfaces
domain=pxe-server
# DHCP range-leases
dhcp-range= ens192,192.168.1.220,192.168.1.230,255.255.255.0,1h
# PXE
dhcp-boot=pxelinux.0,pxeserver,192.168.1.70
# Gateway
dhcp-option=3,192.168.1.1
# DNS
dhcp-option=6,92.168.1.1, 8.8.8.8
server=8.8.4.4
# Broadcast Address
dhcp-option=28,10.0.0.255
# NTP Server
dhcp-option=42,0.0.0.0

pxe-prompt="Press F8 for menu.", 60
pxe-service=x86PC, "Install CentOS 7 from network server 192.168.1.70", pxelinux
enable-tftp
tftp-root=/var/lib/tftpboot
```

Step: 3
**Edit and Config tftp server (/etc/xinetd.d/tftp)**
```
vi /etc/xinetd.d/tftp
service tftp
{
 socket_type = dgram
 protocol    = udp
 wait        = yes
 user        = root
 server      = /usr/sbin/in.tftpd
 server_args = -s /var/lib/tftpboot
 disable     = no
 per_source  = 11
 cps         = 100 2
 flags       = IPv4
}
```

Step: 4
**All the network boot related files are to be placed in tftp root directory “/var/lib/tftpboot”**

**Run the following commands to copy required network boot files in ‘/var/lib/tftpboot/’**
```
cp -v /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot
cp -v /usr/share/syslinux/menu.c32 /var/lib/tftpboot
cp -v /usr/share/syslinux/memdisk /var/lib/tftpboot
cp -v /usr/share/syslinux/mboot.c32 /var/lib/tftpboot
cp -v /usr/share/syslinux/chain.c32 /var/lib/tftpboot
mkdir /var/lib/tftpboot/pxelinux.cfg
mkdir /var/lib/tftpboot/networkboot
```

Step: 5
**Mount CentOS 7.x ISO file and copy its contents to local ftp server**

Step: 6
**Copy Kernel file (vmlimz) and initrd file from mounted iso file to ‘/var/lib/tftpboot/networkboot/’**
```
cp /mnt/images/pxeboot/vmlinuz /var/lib/tftpboot/networkboot/
cp /mnt/images/pxeboot/initrd.img /var/lib/tftpboot/networkboot/
```

Step: 7
**Create kickStart & PXE menu file.**

**Create /tftpboot/pxelinux.cnf/default configuration file. The following is my default file configuration:**
```
default menu.c32
prompt 0
timeout 30
MENU TITLE #############  PXE Menu  ##############
label 1
MENU LABEL ^1) CentOS 7_X64
KERNEL /networkboot/centos7/vmlinuz
APPEND initrd=/networkboot/centos7/initrd.img inst.repo=ftp://192.168.1.70/.Pxe/centos7 ks=ftp://192.168.1.70/.Pxe/centos7/centos7.cfg

label 2
menu label ^2) Ubuntu 18.04
KERNEL /networkboot/ubuntud/linux
APPEND vga=normal initrd=/networkboot/ubuntud/initrd.gz ramdisk_size=16432 root=/dev/rd/0 rw  --
ks=ftp://192.168.1.70/.Pxe/ubuntud/ubuntu.cfg

label 3
menu label ^3) Local Computer
LOCALBOOT 0
```

---

# Kickstart File Syntax









