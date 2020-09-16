**##################################################################################**
# SETTING UP DNS SERVER ON CENTOS 7 STEP BY STEP !!!

## Pre-requisites

#### DNS Server Details:
```
Operating System     : CentOS Linux release 7.7.1908 (Core)
Hostname             : dns-server
IP Address           : 192.168.0.60/24
```
#### Client Details:
```
Operating System     : CentOS Linux release 7.6.1810 (Core)  
Hostname             : dns-client
IP Address           : 192.168.0.61/24
```
**###################################################################################**
# `DNS SERVER`

## DNS Server Installation

##### First we create sudo user `sysadmin` passwordless.
```
[root@dns-server ~]# adduser sysadmin
[root@dns-server ~]# passwd sysadmin
[root@dns-server ~]# visudo
```
###### Sudo user entry image screenshot
<img src="/DNS-Server/img/DNS-1.jfif" width="600" hight="100">

##### Switch to sysadmin user and provide the `static IP address` to DNS Server.
```
[root@dns-server ~]# su -l sysadmin
[sysadmin@dns-server ~]$ sudo cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
TYPE=Ethernet
BOOTPROTO=static
NAME=enp0s3
IPADDR=192.168.0.60
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DEVICE=enp0s3
ONBOOT=yes
[sysadmin@dns-server ~]$
```
##### Set the `hostname` in the network file.
```
[sysadmin@dns-server ~]$ sudo cat /etc/sysconfig/network
# Created by anaconda

HOSTNAME=dns1.godiwal.com
[sysadmin@dns-server ~]$
```
###### Hostname entry image screenshot
<img src="Images/DNS_Image_2.JPG" width="600" hight="100">

##### Add entry `"Server_IP   Your_Domain_Name"` in `"/etc/hosts"` file 
```
[sysadmin@dns-server ~]$ sudo cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.0.60   dns1.godiwal.com
[sysadmin@dns-server ~]$
```
###### Hostname entry image screenshot
<img src="Images/DNS_Image_3.JPG" width="600" hight="100">

##### Add `nameserver` in `"/etc/resolv.conf"` file for resolve the nameserver.
```
[sysadmin@dns-server ~]$ sudo cat /etc/resolv.conf
nameserver 192.168.0.60
```
###### Nameserver entry image screenshot
<img src="Images/DNS_Image_4.JPG" width="600" hight="100">

##### Install `bind`package and all depencies.
```
[sysadmin@dns-server ~]$ sudo yum install bind* -y
[sysadmin@dns-server ~]$ sudo cp /etc/named.conf /etc/named.conf_orig
[sysadmin@dns-server ~]$ sudo vim /etc/named.conf
```
##### Add/Change the lines in `"/etc/named.conf"`.
```
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
        listen-on port 53 { 192.168.0.60; };   ### Master DNS IP ###
#       listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
```
###### Hostname Entry Image Screenshot
<img src="Images/DNS_Image_5.JPG" width="600" hight="100">

##### Take backup of `named.rfc1912.zones` and add/change the entry.
```
[sysadmin@dns-server ~]$ sudo cp /etc/named.rfc1912.zones /etc/named.rfc1912.zones_orig
```
##### After changes file looking like this  
```
[sysadmin@dns-server ~]$ sudo cat /etc/named.rfc1912.zones
// named.rfc1912.zones:
//
// Provided by Red Hat caching-nameserver package
//
// ISC BIND named zone configuration for zones recommended by
// RFC 1912 section 4.1 : localhost TLDs and address zones
// and http://www.ietf.org/internet-drafts/draft-ietf-dnsop-default-local-zones-02.txt
// (c)2007 R W Franks
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

zone "godiwal.com" IN {
        type master;
        file "forward.zone";
        allow-update { none; };

};

zone "localhost" IN {
        type master;
        file "named.localhost";
        allow-update { none; };
};

zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" IN {
        type master;
        file "named.loopback";
        allow-update { none; };
};

zone "0.168.192.in-addr.arpa" IN {
        type master;
        file "reverse.zone";
        allow-update { none; };
};

zone "0.in-addr.arpa" IN {
        type master;
        file "named.empty";
        allow-update { none; };
};

[sysadmin@dns-server ~]$
```
###### `/etc/named.rfc1912.zones` entry image screenshot
<img src="Images/DNS_Image_6.JPG" width="600" hight="100">

###### Check the all file in this directory `/var/named/`.
```
[sysadmin@dns-server ~]$ sudo ls -l /var/named/
total 16
drwxr-x---. 7 root  named   61 May 11 08:07 chroot
drwxr-x---. 7 root  named   61 May 11 08:07 chroot_sdb
drwxrwx---. 2 named named    6 Apr  7 20:11 data
drwxrwx---. 2 named named    6 Apr  7 20:11 dynamic
drwxrwx---. 2 root  named    6 Apr  1 07:46 dyndb-ldap
-rw-r-----. 1 root  named 2253 Apr  5  2018 named.ca
-rw-r-----. 1 root  named  152 Dec 15  2009 named.empty
-rw-r-----. 1 root  named  152 Jun 21  2007 named.localhost
-rw-r-----. 1 root  named  168 Dec 15  2009 named.loopback
drwxrwx---. 2 named named    6 Apr  7 20:11 slaves
[sysadmin@dns-server ~]$
```
##### Copy named.localhost for create forward lookup zone and then do some changes accordingly.
```
[sysadmin@dns-server ~]$ sudo cp /var/named/named.localhost /var/named/forward.zone
# After changes the forward.zone file looking like this.
[sysadmin@dns-server ~]$ sudo cat /var/named/forward.zone
$TTL 1D
@       IN SOA  dns1.godiwal.com. root.dns1.godiwal.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum


        IN      NS      dns1.godiwal.com.
dns1    IN      A       192.168.0.60
[sysadmin@dns-server ~]$
```
###### `forward.zone` entry image screenshot
<img src="Images/DNS_Image_7.JPG" width="600" hight="100">

##### Copy named.localhost for create reverse lookup zone and then do some changes accordingly.
```
[sysadmin@dns-server ~]$ sudo cp /var/named/named.localhost /var/named/reverse.zone
# After changes the reverse.zone file looking like this.
[sysadmin@dns-server ~]$ sudo cat /var/named/reverse.zone
$TTL 1D
@       IN SOA  dns1.godiwal.com. root.dns1.godiwal.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum


        IN      NS      dns1.godiwal.com.
60      IN      PTR     dns1.godiwal.com.
[sysadmin@dns-server ~]$
```

###### `reverse.zone` entry image screenshot
<img src="Images/DNS_Image_8.JPG" width="600" hight="100">

##### We have to check `forward.zone` and `reverse.zone` file `group permission` and change with `named` group.
```
[sysadmin@dns-server ~]$ sudo ls -l /var/named/
total 24
drwxr-x---. 7 root  named   61 May 11 08:07 chroot
drwxr-x---. 7 root  named   61 May 11 08:07 chroot_sdb
drwxrwx---. 2 named named    6 Apr  7 20:11 data
drwxrwx---. 2 named named    6 Apr  7 20:11 dynamic
drwxrwx---. 2 root  named    6 Apr  1 07:46 dyndb-ldap
-rw-r-----. 1 root  root   434 May 11 09:29 forward.zone
-rw-r-----. 1 root  named 2253 Apr  5  2018 named.ca
-rw-r-----. 1 root  named  152 Dec 15  2009 named.empty
-rw-r-----. 1 root  named  152 Jun 21  2007 named.localhost
-rw-r-----. 1 root  named  168 Dec 15  2009 named.loopback
-rw-r-----. 1 root  root   439 May 11 09:37 reverse.zone
drwxrwx---. 2 named named    6 Apr  7 20:11 slaves
[sysadmin@dns-server ~]$
```
##### Without change both file permission looking like this.
<img src="Images/DNS_Image_9.JPG" width="600" hight="100">

##### So we will run below commands for that.
```
[sysadmin@dns-server ~]$ sudo chgrp named /var/named/forward.zone
[sysadmin@dns-server ~]$ sudo chgrp named /var/named/reverse.zone
[sysadmin@dns-server ~]$ sudo ls -l /var/named/
total 24
drwxr-x---. 7 root  named   61 May 11 08:07 chroot
drwxr-x---. 7 root  named   61 May 11 08:07 chroot_sdb
drwxrwx---. 2 named named    6 Apr  7 20:11 data
drwxrwx---. 2 named named    6 Apr  7 20:11 dynamic
drwxrwx---. 2 root  named    6 Apr  1 07:46 dyndb-ldap
-rw-r-----. 1 root  named  434 May 11 09:29 forward.zone
-rw-r-----. 1 root  named 2253 Apr  5  2018 named.ca
-rw-r-----. 1 root  named  152 Dec 15  2009 named.empty
-rw-r-----. 1 root  named  152 Jun 21  2007 named.localhost
-rw-r-----. 1 root  named  168 Dec 15  2009 named.loopback
-rw-r-----. 1 root  named  439 May 11 09:37 reverse.zone
drwxrwx---. 2 named named    6 Apr  7 20:11 slaves
[sysadmin@dns-server ~]$
```
##### After that both file permission looking like this.
<img src="Images/DNS_Image_10.JPG" width="600" hight="100">

##### Check all file configuration is perfect or not. If you will get same output so everything is fine. 
```
[sysadmin@dns-server ~]$ sudo named-checkconf /etc/named.conf
[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ sudo named-checkzone unixmen.local /var/named/forward.zone
zone unixmen.local/IN: loaded serial 0
OK
[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ sudo named-checkzone unixmen.local /var/named/reverse.zone
zone unixmen.local/IN: loaded serial 0
OK
[sysadmin@dns-server ~]$
```
##### Add DNS UPD and TCP port in Firewall.
```
[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ sudo firewall-cmd --add-port=53/tcp --permanent
success
[sysadmin@dns-server ~]$ sudo firewall-cmd --add-port=53/udp --permanent
success
[sysadmin@dns-server ~]$ sudo firewall-cmd --reload
success
[sysadmin@dns-server ~]$
```
##### Start, enable and check the status of the DNS Service.
```
[sysadmin@dns-server ~]$ sudo systemctl start named
[sysadmin@dns-server ~]$ sudo systemctl enable named
Created symlink from /etc/systemd/system/multi-user.target.wants/named.service to /usr/lib/systemd/system/named.service.
[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ sudo systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-05-11 09:58:39 IST; 10s ago
  Process: 15899 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 15897 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 15901 (named)
   CGroup: /system.slice/named.service
           └─15901 /usr/sbin/named -u named -c /etc/named.conf

May 11 09:58:39 dns-server named[15901]: network unreachable resolving './DNSKEY/IN': 2001:503:c27::2:30#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './NS/IN': 2001:503:c27::2:30#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './DNSKEY/IN': 2001:500:2f::f#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './NS/IN': 2001:500:2f::f#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './NS/IN': 2001:500:1::53#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
May 11 09:58:39 dns-server named[15901]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
May 11 09:58:39 dns-server named[15901]: managed-keys-zone: Key 20326 for zone . acceptance timer complete: key now trusted
May 11 09:58:39 dns-server named[15901]: resolver priming query complete
[sysadmin@dns-server ~]$
```
##### When you run below command so you will see in end of the output `"SERVER: 192.168.0.60#53(192.168.0.60)"`. It's mean your DNS Server perfectly working.
```
[sysadmin@dns-server ~]$ dig

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10119
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;.                              IN      NS

;; ANSWER SECTION:
.                       518271  IN      NS      h.root-servers.net.
.                       518271  IN      NS      f.root-servers.net.
.                       518271  IN      NS      a.root-servers.net.
.                       518271  IN      NS      k.root-servers.net.
.                       518271  IN      NS      j.root-servers.net.
.                       518271  IN      NS      e.root-servers.net.
.                       518271  IN      NS      m.root-servers.net.
.                       518271  IN      NS      l.root-servers.net.
.                       518271  IN      NS      b.root-servers.net.
.                       518271  IN      NS      d.root-servers.net.
.                       518271  IN      NS      g.root-servers.net.
.                       518271  IN      NS      c.root-servers.net.
.                       518271  IN      NS      i.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.     518271  IN      A       198.41.0.4
b.root-servers.net.     518271  IN      A       199.9.14.201
c.root-servers.net.     518271  IN      A       192.33.4.12
d.root-servers.net.     518271  IN      A       199.7.91.13
e.root-servers.net.     518271  IN      A       192.203.230.10
f.root-servers.net.     518271  IN      A       192.5.5.241
g.root-servers.net.     518271  IN      A       192.112.36.4
h.root-servers.net.     518271  IN      A       198.97.190.53
i.root-servers.net.     518271  IN      A       192.36.148.17
j.root-servers.net.     518271  IN      A       192.58.128.30
k.root-servers.net.     518271  IN      A       193.0.14.129
l.root-servers.net.     518271  IN      A       199.7.83.42
m.root-servers.net.     518271  IN      A       202.12.27.33
a.root-servers.net.     518271  IN      AAAA    2001:503:ba3e::2:30
b.root-servers.net.     518271  IN      AAAA    2001:500:200::b
c.root-servers.net.     518271  IN      AAAA    2001:500:2::c
d.root-servers.net.     518271  IN      AAAA    2001:500:2d::d
e.root-servers.net.     518271  IN      AAAA    2001:500:a8::e
f.root-servers.net.     518271  IN      AAAA    2001:500:2f::f
g.root-servers.net.     518271  IN      AAAA    2001:500:12::d0d
h.root-servers.net.     518271  IN      AAAA    2001:500:1::53
i.root-servers.net.     518271  IN      AAAA    2001:7fe::53
j.root-servers.net.     518271  IN      AAAA    2001:503:c27::2:30
k.root-servers.net.     518271  IN      AAAA    2001:7fd::1
l.root-servers.net.     518271  IN      AAAA    2001:500:9f::42
m.root-servers.net.     518271  IN      AAAA    2001:dc3::35

;; Query time: 0 msec
;; SERVER: 192.168.0.60#53(192.168.0.60)
;; WHEN: Mon May 11 10:00:48 IST 2020
;; MSG SIZE  rcvd: 811

[sysadmin@dns-server ~]$
```
##### Check below screenshot.
<img src="Images/DNS_Image_11.JPG" width="600" hight="100">

##### Now we'll check our Domain Name with dig and nslookup command.
```
[sysadmin@dns-server ~]$ dig dns1.godiwal.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>> dns1.godiwal.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50473
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;dns1.godiwal.com.              IN      A

;; ANSWER SECTION:
dns1.godiwal.com.       86400   IN      A       192.168.0.60

;; AUTHORITY SECTION:
godiwal.com.            86400   IN      NS      dns1.godiwal.com.

;; Query time: 0 msec
;; SERVER: 192.168.0.60#53(192.168.0.60)
;; WHEN: Mon May 11 10:03:51 IST 2020
;; MSG SIZE  rcvd: 75

[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ nslookup dns1.godiwal.com
Server:         192.168.0.60
Address:        192.168.0.60#53

Name:   dns1.godiwal.com
Address: 192.168.0.60

[sysadmin@dns-server ~]$
[sysadmin@dns-server ~]$ nslookup 192.168.0.60
60.0.168.192.in-addr.arpa       name = dns1.godiwal.com.

[sysadmin@dns-server ~]$

```
##### Check below screenshot respectively.
<img src="Images/DNS_Image_12.JPG" width="600" hight="100">
<img src="Images/DNS_Image_13.JPG" width="600" hight="100">
<img src="Images/DNS_Image_14.JPG" width="600" hight="100">

**###################################################################################**
# `CLIENT HOST`

##### Check Client to Server connectivity with ping command.
```
[root@dns-client ~]# ping 192.168.0.60
PING 192.168.0.60 (192.168.0.60) 56(84) bytes of data.
64 bytes from 192.168.0.60: icmp_seq=1 ttl=64 time=1.07 ms
64 bytes from 192.168.0.60: icmp_seq=2 ttl=64 time=0.417 ms
64 bytes from 192.168.0.60: icmp_seq=3 ttl=64 time=0.414 ms
64 bytes from 192.168.0.60: icmp_seq=4 ttl=64 time=0.465 ms
[root@dns-client ~]#
```
##### Install dig command for checking our DNS Server from Client Host. 
```
[root@dns-client ~]# yum install bind-utils -y
[root@dns-client ~]# dig dns1.vardhan.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>> dns1.vardhan.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38530
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 7

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;dns1.vardhan.com.              IN      A

;; ANSWER SECTION:
dns1.vardhan.com.       3600    IN      A       69.172.201.153

;; AUTHORITY SECTION:
vardhan.com.            172800  IN      NS      buy.internettraffic.com.
vardhan.com.            172800  IN      NS      sell.internettraffic.com.

;; ADDITIONAL SECTION:
buy.internettraffic.com. 55969  IN      A       52.89.18.51
buy.internettraffic.com. 55969  IN      A       54.152.30.238
buy.internettraffic.com. 55969  IN      A       35.211.36.175
sell.internettraffic.com. 55969 IN      A       34.247.146.68
sell.internettraffic.com. 55969 IN      A       18.217.16.210
sell.internettraffic.com. 55969 IN      A       34.90.60.247

;; Query time: 486 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Mon May 11 10:20:38 IST 2020
;; MSG SIZE  rcvd: 210

[root@dns-client ~]#
```
##### It will not show your DNS Server details. So you have to resolve your DNS Server. 
```
[root@dns-client ~]# cat /etc/resolv.conf
nameserver 192.168.0.60
[root@dns-client ~]#
[root@dns-client ~]# dig dns1.godiwal.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>> dns1.godiwal.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39764
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;dns1.godiwal.com.              IN      A

;; ANSWER SECTION:
dns1.godiwal.com.       86400   IN      A       192.168.0.60

;; AUTHORITY SECTION:
godiwal.com.            86400   IN      NS      dns1.godiwal.com.

;; Query time: 1 msec
;; SERVER: 192.168.0.60#53(192.168.0.60)
;; WHEN: Mon May 11 10:24:02 IST 2020
;; MSG SIZE  rcvd: 75

[root@dns-client ~]#
[root@dns-client ~]# nslookup dns1.godiwal.com
Server:         192.168.0.60
Address:        192.168.0.60#53

Name:   dns1.godiwal.com
Address: 192.168.0.60

[root@dns-client ~]#
[root@dns-client ~]# nslookup 192.168.0.60
60.0.168.192.in-addr.arpa       name = dns1.godiwal.com.

[root@dns-client ~]#

[root@dns-client ~]# ping dns1.godiwal.com
PING dns1.godiwal.com (192.168.0.60) 56(84) bytes of data.
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=1 ttl=64 time=0.326 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=2 ttl=64 time=0.813 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=3 ttl=64 time=0.393 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=4 ttl=64 time=0.817 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=5 ttl=64 time=0.849 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=6 ttl=64 time=1.25 ms
64 bytes from dns1.godiwal.com (192.168.0.60): icmp_seq=7 ttl=64 time=3.30 ms
^C
[root@dns-client ~]#
```
##### Check below screenshot respectively.
<img src="Images/DNS_Image_15.JPG" width="600" hight="100">
<img src="Images/DNS_Image_16.JPG" width="600" hight="100">
<img src="Images/DNS_Image_17.JPG" width="600" hight="100">

# `SUCCESSFULLY COMPLETE`
**###################################################################################**
