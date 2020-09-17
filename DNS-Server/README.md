**##################################################################################**
# SETTING UP DNS SERVER ON CENTOS 7 STEP BY STEP !!!

<img src="/DNS-Server/img/DNS.JPG" width="400" hight="500">

DNS, stands for Domain Name System, translates hostnames or URLs into IP addresses. For example, if we type www.google.com in browser, the DNS server translates the domain name into its associated ip address. Since the IP addresses are hard to remember all time, DNS servers are used to translate the hostnames like www.google.com to 8.xxx.xx.xxx. So it makes easy to remember the domain names instead of its IP address.

## Pre-requisites

For the purpose of this tutorial, I will be using three nodes. One will be acting as Master DNS server, the second system will be acting as Secondary DNS, and the third will be our DNS client. Here are my three systems details.

#### Primary (Master) DNS Server Details::
```
Operating System     : CentOS Linux release 7.7.1908 (Core)
Hostname             : masterdns.sethi.com
IP Address           : 192.168.0.10/24
```
#### Secondary (Slave) DNS Server Details:
```
Operating System     : CentOS Linux release 7.6.1810 (Core)  
Hostname             : secondarydns.sethi.com
IP Address           : 192.168.0.11/24
```
### Client Details:
```bash
Operating System     : CentOS Linux release 7.6.1810 (Core)
Hostname             : client.sethi.com
IP Address           : 192.168.0.12/24
```

**###################################################################################**
# `DNS SERVER`

## Setup Primary (Master) DNS Server

### Install bind packages on your server
```bash
yum install bind bind-utils -y
```

**1. Configure DNS Server**
`Edit ‘/etc/named.conf’ file.`

**Add the lines as shown in bold:**
```bash
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
    listen-on port 53 { 127.0.0.1; 192.168.0.10;}; ### Master DNS IP ###
#    listen-on-v6 port 53 { ::1; };
    directory     "/var/named";
    dump-file     "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { localhost; 192.168.0.0/24;}; ### IP Range ###
    allow-transfer{ localhost; 192.168.0.11; };   ### Slave DNS IP ###

    /* 
     - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
     - If you are building a RECURSIVE (caching) DNS server, you need to enable 
       recursion. 
     - If your recursive DNS server has a public IP address, you MUST enable access 
       control to limit queries to your legitimate users. Failing to do so will
       cause your server to become part of large scale DNS amplification 
       attacks. Implementing BCP38 within your network would greatly
       reduce such attack surface 
    */
    recursion yes;

    dnssec-enable yes;
    dnssec-validation yes;
    dnssec-lookaside auto;

    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
    type hint;
    file "named.ca";
};

zone "sethi.com" IN {
type master;
file "forward.sethi";
allow-update { none; };
};
zone "0.168.192.in-addr.arpa" IN {
type master;
file "reverse.sethi";
allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

2. Create Zone files
Create forward and reverse zone files which we mentioned in the ‘/etc/named.conf’ file.

2.1 Create Forward Zone
Create forward.sethi file in the ‘/var/named’ directory.
```bash
vi /var/named/forward.sethi
```
Add the following lines:
```bash
$TTL 86400
@   IN  SOA     masterdns.sethi.com. root.sethi.com. (
        1001        ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
;Name Server Information
@       IN  NS          masterdns.sethi.com.

;IP address of Name Server
@       IN  A           192.168.0.10

;Mail exchanger
sethi.com. IN  MX 10   mail.sethi.com.

;A - Record HostName To IP Address
masterdns       IN  A   192.168.0.10

```

2.2 Create Reverse Zone
Create reverse.sethi file in the ‘/var/named’ directory.

vi /var/named/reverse.sethi

Add the following lines:
```bash
$TTL 86400
@   IN  SOA     masterdns.sethi.com. root.sethi.com. (
        1001        ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
;Name Server Information
@       IN  NS          masterdns.sethi.com.

;Reverse lookup for Name Server
@       IN  PTR         sethi.com.

masterdns       IN  A   192.168.0.10

;PTR Record IP address to HostName
10     IN  PTR         masterdns.sethi.com.

```

3. Firewall Configuration
We must allow the DNS service default port 53 through firewall.
```bash
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
```

4. Restart Firewall
```bash
firewall-cmd --reload
```

5. Configuring Permissions, Ownership, and SELinux
Run the following commands one by one:
```bash
chgrp named -R /var/named
chown -v root:named /etc/named.conf
restorecon -rv /var/named
restorecon /etc/named.conf
```

6. Test DNS configuration and zone files for any syntax errors
Check DNS default configuration file:
```bash
named-checkconf /etc/named.conf
```
If it returns nothing, your configuration file is valid.

### Check Forward zone:
```bash
named-checkzone sethi.com /var/named/forward.sethi
```
Sample output:
```bash
zone sethi.com/IN: loaded serial 1002
OK
```

### Check reverse zone:
```bash
named-checkzone sethi.com /var/named/reverse.sethi 
```
Sample Output:
```bash
zone sethi.com/IN: loaded serial 1001
OK
```

7. Start the DNS service
Enable and start DNS service:
```bash
systemctl start named
systemctl enable named
```

##### When you run below command so you will see in end of the output `"SERVER: 192.168.0.10#53(192.168.0.10)"`. It's mean your DNS Server perfectly working.
```
[root@masterdns ~]# dig
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10789
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 13, ADDITIONAL: 7

;; QUESTION SECTION:
;.                              IN      NS

;; ANSWER SECTION:
.                       5       IN      NS      c.root-servers.net.
.                       5       IN      NS      m.root-servers.net.
.                       5       IN      NS      l.root-servers.net.
.                       5       IN      NS      j.root-servers.net.
.                       5       IN      NS      h.root-servers.net.
.                       5       IN      NS      g.root-servers.net.
.                       5       IN      NS      a.root-servers.net.
.                       5       IN      NS      k.root-servers.net.
.                       5       IN      NS      f.root-servers.net.
.                       5       IN      NS      i.root-servers.net.
.                       5       IN      NS      b.root-servers.net.
.                       5       IN      NS      e.root-servers.net.
.                       5       IN      NS      d.root-servers.net.

;; AUTHORITY SECTION:
.                       5       IN      NS      m.root-servers.net.
.                       5       IN      NS      l.root-servers.net.
.                       5       IN      NS      j.root-servers.net.
.                       5       IN      NS      h.root-servers.net.
.                       5       IN      NS      g.root-servers.net.
.                       5       IN      NS      a.root-servers.net.
.                       5       IN      NS      k.root-servers.net.
.                       5       IN      NS      f.root-servers.net.
.                       5       IN      NS      i.root-servers.net.
.                       5       IN      NS      b.root-servers.net.
.                       5       IN      NS      e.root-servers.net.
.                       5       IN      NS      d.root-servers.net.
.                       5       IN      NS      c.root-servers.net.

;; ADDITIONAL SECTION:
c.root-servers.net.     5       IN      A       192.33.4.12
g.root-servers.net.     5       IN      A       192.112.36.4
a.root-servers.net.     5       IN      A       198.41.0.4
f.root-servers.net.     5       IN      A       192.5.5.241
b.root-servers.net.     5       IN      A       199.9.14.201
e.root-servers.net.     5       IN      A       192.203.230.10
d.root-servers.net.     5       IN      A       199.7.91.13

;; Query time: 9 msec
;; SERVER: 192.168.0.10#53(192.168.0.10)
;; WHEN: Thu Sep 17 08:51:10 IST 2020
;; MSG SIZE  rcvd: 509
```

##### Now we'll check our Domain Name with dig and nslookup command.
```
[root@masterdns ~]# dig masterdns.sethi.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> masterdns.sethi.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54439
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;masterdns.sethi.com.           IN      A

;; ANSWER SECTION:
masterdns.sethi.com.    86400   IN      A       192.168.0.10

;; AUTHORITY SECTION:
sethi.com.              86400   IN      NS      masterdns.sethi.com.

;; Query time: 0 msec
;; SERVER: 192.168.0.10#53(192.168.0.10)
;; WHEN: Thu Sep 17 08:58:01 IST 2020
;; MSG SIZE  rcvd: 78
```
### Nslookup
```
[root@masterdns ~]# nslookup masterdns.sethi.com
Server:         192.168.0.10
Address:        192.168.0.10#53

Name:   masterdns.sethi.com
Address: 192.168.0.10
```
```
[root@masterdns ~]$ nslookup 192.168.0.10
10.0.168.192.in-addr.arpa       name = masterdns.sethi.com.
```

**###################################################################################**
# `CLIENT HOST`

##### Check Client to Server connectivity with ping command.
```
[root@dns-client ~]# ping 192.168.0.10
PING 192.168.0.60 (192.168.0.10) 56(84) bytes of data.
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

# `SUCCESSFULLY COMPLETE`
**###################################################################################**
