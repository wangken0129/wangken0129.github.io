---
title: OCP-4.11.7_on_Bare-metal
description: OCP-4.11.7_on_Bare-metal
slug: OCP-4.11.7_on_Bare-metal
date: 2023-09-23T08:18:45+08:00
categories:
    - Lab Category
tags:
    - Openshift
    - Redhat
    - Kubernetes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Install Openshift v4.11.7 on Bare-metal

Ken Wang 2022/10/03

Ken Wang 2022/11/23 v4.11.9



## 參考連結

https://docs.openshift.com/container-platform/4.11/welcome/index.html

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.11/html/installing/installing-on-bare-metal#installing-bare-metal

https://docs.openshift.com/container-platform/4.11/installing/installing_bare_metal/installing-bare-metal.html#installing-bare-metal

CoreOS:
https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.11/4.11.2/
https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.11/4.11.9/rhcos-4.11.9-x86_64-live.x86_64.iso

OC Client:
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.11.9/openshift-client-linux.tar.gz

https://docs.openshift.com/container-platform/4.11/post_installation_configuration/index.html

https://github.com/I8C/installing-openshift-on-single-vmware-esxi-host/blob/master/installBastion.sh


僅供參考 https://github.com/alanadiprastyo/openshift-4.6

automated

https://cloud.redhat.com/blog/fully-automated-openshift-deployments-with-vmware-vsphere



## 架構

針對各版本的coreOS、Openshift-install 都要更新

| 主機名稱              | IP            | Mac               | 版本        | 功能       | 磁碟空間   |
| :-------------------- | ------------- | ----------------- | ----------- | ---------- | ---------- |
| bastion.ken.lab       | 192.168.33.21 | 00:50:56:91:7c:24 | RHEL 9.0    | management | 100G、200G |
| bootstrap.ocp.ken.lab | 192.168.33.22 | 00:50:56:91:47:e9 | core 4.11-2 | bootstrap  | 100G       |
| master1.ocp.ken.lab   | 192.168.33.23 | 00:50:56:91:88:f4 | core 4.11-2 | master     | 100G       |
| master2.ocp.ken.lab   | 192.168.33.24 | 00:50:56:91:27:ed | core 4.11-2 | master     | 100G       |
| master3.ocp.ken.lab   | 192.168.33.25 | 00:50:56:91:08:33 | core 4.11-2 | master     | 100G       |
| worker1.ocp.ken.lab   | 192.168.33.26 | 00:50:56:91:c6:29 | core 4.11-2 | worker     | 100G       |
| worker2.ocp.ken.lab   | 192.168.33.27 | 00:50:56:91:d7:b4 | core 4.11-2 | worker     | 100G       |
| infra1.ocp.ken.lab    | 192.168.33.31 | 00:50:56:91:14:ab | core 4.11-2 | infra node | 100G       |
| infra2.ocp.ken.lab    | 192.168.33.32 | 00:50:56:91:0a:53 | core 4.11-2 | infra node | 100G       |
| infra3.ocp.ken.lab    | 192.168.33.33 | 00:50:56:91:78:05 | core 4.11-2 | infra node | 100G       |

| 主機名稱        | 使用者帳密     | 安裝目錄        |      |
| --------------- | -------------- | --------------- | ---- |
| bastion.ken.lab | devop / redhat | /home/devop/ocp |      |



## 安裝Bastion



### 安裝RHEL 9.0  on ESXi 

https://kb.vmware.com/s/article/57122

1. 用iso 檔開機進入安裝引導畫面
   ![image-20221003152533576](https://kenkenny.synology.me:5543/images/2023/09/image-20221003152533576.png)

2. 設定磁碟、註冊、安裝軟體
   - 磁碟配置
     / >> 65GiB
     /home >> 29GiB
     /boot >> 500MiB
     /boot/efi >> 1GiB
   - 軟體安裝 使用GUI畫面
     Server With GUI 
   - 註冊
     ken.wang@infotech.com.tw

3. 開始安裝，安裝完畢後進入OS
   ![image-20221003153827189](https://kenkenny.synology.me:5543/images/2023/09/image-20221003153827189.png)

4. 設定sudo 

   ```
   [root@bastion ~]# echo "devop    ALL=(ALL)       ALL" >> /etc/sudoers.d/devop
   ```

---

### 設定 SSH Key

1. 產生ssh keygen -- devop user
   用來登入各節點使用

   ```
   [devop@bastion ~]$ ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
   Generating public/private ed25519 key pair.
   Created directory '/home/devop/.ssh'.
   Your identification has been saved in /home/devop/.ssh/id_ed25519
   Your public key has been saved in /home/devop/.ssh/id_ed25519.pub
   The key fingerprint is:
   SHA256:G91wVoISB5qA881zCUi71a2hHj2Y+3zPD6RJQW5+edM devop@bastion.ocp.ken.lab
   The key's randomart image is:
   +--[ED25519 256]--+
   |   oo.  ooo.. .  |
   |  o .o.+.=.  o   |
   |   o.o+.oo* o    |
   |    .o+=o* * . . |
   |    . =oS + = o E|
   |     . o = = . . |
   |      o . o .    |
   |       o  .. .   |
   |        o. .o..  |
   +----[SHA256]-----+
   
   ```

2. 查看public key >> 之後會寫入install-config.yaml

   ```
   [devop@bastion ~]$ cat .ssh/id_ed25519.pub 
   ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIN1Z2jXmGnPWGAVrtltN9A2q5D8QtJlRvhvdB2QXoZG devop@bastion.ocp.ken.lab
   ```

3. 加入私密金鑰到ssh agent

   ```
   [devop@bastion ~]$ ssh-add .ssh/id_ed25519
   Identity added: .ssh/id_ed25519 (devop@bastion.ocp.ken.lab)
   ```

   ---

### 安裝OC CLI

1. 從此處下載OC Client
   https://access.redhat.com/downloads/content/290/ver=4.11/rhel---8/4.11.7/x86_64/product-software
   https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.11.9/openshift-client-linux.tar.gz
   
2. 安裝OC Client

   ```
   [devop@bastion oc]$ pwd
   /home/devop/oc
   [devop@bastion oc]$ tar xvzf oc-4.11.7-linux.tar.gz 
   README.md
   kubectl
   oc
   [devop@bastion oc]$ sudo cp oc kubectl /usr/local/bin/
   [sudo] password for devop: 
   [devop@bastion oc]$ oc
   OpenShift Client
   
   This client helps you develop, build, deploy, and run your applications on any
   OpenShift or Kubernetes cluster. It also includes the administrative
   ...
   ```

   啟用 tab 自動完成功能

   ```
   [devop@bastion ocp]$ oc completion bash > oc_bash_completion
   [devop@bastion ocp]$ sudo cp oc_bash_completion /etc/bash_completion.d/
   [sudo] password for devop: 
   ## 重新啟動terminal
   ```

   ---

   

### 安裝DHCP Server (optional)

1. 安裝dhcpd

   ```
   [root@bastion ~]# yum install dhcp-server
   ```

2. 修改dhcpd設定檔 /etc/dhcp/dhcpd.conf

   ```
   [root@bastion dhcp]# cat dhcpd.conf 
   #
   # DHCP Server Configuration file.
   #   see /usr/share/doc/dhcp-server/dhcpd.conf.example
   #   see dhcpd.conf(5) man page
   #
   # dhcpd.conf
   # 2022.10.11 ken wang
   default-lease-time 600;
   max-lease-time 7200;
   option broadcast-address 192.168.33.255;
   subnet 192.168.33.0 netmask 255.255.255.0 {}
   ## gateway
   option routers 192.168.33.251;
   option domain-name "ocp.ken.lab";
   option domain-name-servers 192.168.33.21, 168.95.1.1;
   
   # Use this to send dhcp log messages to a different log file (you also
   # have to hack syslog.conf to complete the redirection).
   log-facility local7;
   
   #  range 192.168.33.22 192.168.33.40;
   host bootstrap {
     hardware ethernet 00:50:56:91:47:e9;
     fixed-address 192.168.33.22;
   }
   host master1 {
     hardware ethernet 00:50:56:91:88:f4;
     fixed-address 192.168.33.23;
   }
   host master2 {
     hardware ethernet 00:50:56:91:27:ed;
     fixed-address 192.168.33.24;
   }
   host master3 {
     hardware ethernet 00:50:56:91:08:33;
     fixed-address 192.168.33.25;
   }
   host worker1 {
     hardware ethernet 00:50:56:91:c6:29;
     fixed-address 192.168.33.26;
   }
   host worker2 {
     hardware ethernet 00:50:56:91:d7:b4;
     fixed-address 192.168.33.27;
   }
   host infra1 {
     hardware ethernet 00:50:56:91:14:ab;
     fixed-address 192.168.33.31;
   }
   host infra2 {
     hardware ethernet 00:50:56:91:0a:53;
     fixed-address 192.168.33.32;
   }
   host infra3 {
     hardware ethernet 00:50:56:91:78:05;
     fixed-address 192.168.33.33;
   }
   ```

3. 啟動dhcp-server測試

   ```
   [root@bastion dhcp]# systemctl status dhcpd
   ● dhcpd.service - DHCPv4 Server Daemon
        Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
        Active: active (running) since Tue 2022-10-11 11:07:02 CST; 10s ago
   
   ```

---

### 安裝DNS Server (named)

1. 安裝named 

   ```
   [root@bastion dhcp]# yum install bind bind-libs bind-chroot bind-utils
   ```

2. 修改named 設定檔 /etc/named.conf

   ```
   [root@bastion data]# cat /etc/named.conf 
   //
   // named.conf
   //
   // Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
   // server as a caching only nameserver (as a localhost DNS resolver only).
   //
   // See /usr/share/doc/bind*/sample/ for example named configuration files.
   //
   
   options {
   	listen-on port 53 { 192.168.33.21; };
   	//listen-on-v6 port 53 { ::1; };
   	directory 	"/var/named";
   	dump-file 	"/var/named/data/cache_dump.db";
   	statistics-file "/var/named/data/named_stats.txt";
   	memstatistics-file "/var/named/data/named_mem_stats.txt";
   	secroots-file	"/var/named/data/named.secroots";
   	recursing-file	"/var/named/data/named.recursing";
   	allow-query     { any; };
   	## 設定轉寄站
   	forwarders {
     192.168.2.12;
     192.168.2.11;
      };
   
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
   
   	dnssec-validation yes;
           dnssec-enable yes;
   
   	managed-keys-directory "/var/named/dynamic";
   	geoip-directory "/usr/share/GeoIP";
   
   	pid-file "/run/named/named.pid";
   	session-keyfile "/run/named/session.key";
   
   	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
   	include "/etc/crypto-policies/back-ends/bind.config";
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
   
   include "/etc/named.rfc1912.zones";
   include "/etc/named.root.key";
   zone "ocp.ken.lab." IN {
   		type master;
   		file "ken.zone";
   		allow-update { none; };
   		allow-query { any; };
   		allow-transfer { none; };
   };
   zone "33.168.192.in-addr.arpa" IN {
   		type master;
   		file "ken.reverse";
   		allow-update { none; };
   		allow-query { any; };
   		allow-transfer { none; };
   };
   ```

3. 修改正向解析 /var/named/ken.zone

   ```
   [root@bastion named]# cat ken.zone 
   $TTL 1W
   @	IN	SOA	bastion.ken.lab.	root (
   			2019070700	; serial
   			3H		; refresh (3 hours)
   			30M		; retry (30 minutes)
   			2W		; expiry (2 weeks)
   			1W )		; minimum (1 week)
   	IN	NS	bastion.ken.lab.
   ;
   ;
   bastion.ken.lab.	IN	A	192.168.33.21
   ;
   helper.ken.lab.		IN	A	192.168.33.21
   helper.ocp.ken.lab.	IN	A	192.168.33.21
   ;
   api.ocp.ken.lab.	IN	A	192.168.33.21 
   api-int.ocp.ken.lab.	IN	A	192.168.33.21 
   ;
   *.apps.ocp.ken.lab.	IN	A	192.168.33.21 
   ;
   bootstrap.ocp.ken.lab.	IN	A	192.168.33.22 
   ;
   master1.ocp.ken.lab.	IN	A	192.168.33.23
   master2.ocp.ken.lab.	IN	A	192.168.33.24
   master3.ocp.ken.lab.	IN	A	192.168.33.25
   ;
   worker1.ocp.ken.lab.	IN	A	192.168.33.26
   worker2.ocp.ken.lab.	IN	A	192.168.33.27 
   ;
   infra1.ocp.ken.lab.	IN	A	192.168.33.31
   infra2.ocp.ken.lab.	IN	A	192.168.33.32
   infra3.ocp.ken.lab.	IN	A	192.168.33.33
   ;
   ;EOF
   ```

4. 修改反解析 /var/named/ken.reverse

   ```
   [root@bastion named]# cat ken.reverse 
   $TTL 10
   @	IN	SOA	bastion.ken.lab.	root (
   			2019070700	; serial
   			3H		; refresh (3 hours)
   			30M		; retry (30 minutes)
   			2W		; expiry (2 weeks)
   			1W )		; minimum (1 week)
   	IN	NS	bastion.ken.lab.
   ;
   ;
   21.33.168.192.in-addr.arpa.	IN	PTR	bastion.ocp.ken.lab. 
   21.33.168.192.in-addr.arpa.	IN	PTR	api.ocp.ken.lab. 
   21.33.168.192.in-addr.arpa.	IN	PTR	api-int.ocp.ken.lab. 
   ;
   22.33.168.192.in-addr.arpa.	IN	PTR	bootstrap.ocp.ken.lab. 
   ;
   23.33.168.192.in-addr.arpa.	IN	PTR	master1.ocp.ken.lab. 
   24.33.168.192.in-addr.arpa.	IN	PTR	master2.ocp.ken.lab. 
   25.33.168.192.in-addr.arpa.	IN	PTR	master3.ocp.ken.lab. 
   ;
   26.33.168.192.in-addr.arpa.	IN	PTR	worker1.ocp.ken.lab. 
   27.33.168.192.in-addr.arpa.	IN	PTR	worker2.ocp.ken.lab. 
   ;
   31.33.168.192.in-addr.arpa.	IN	PTR	infra1.ocp.ken.lab. 
   32.33.168.192.in-addr.arpa.	IN	PTR	infra2.ocp.ken.lab.
   33.33.168.192.in-addr.arpa.	IN	PTR	infra3.ocp.ken.lab.
   ;
   ;EOF
   ```

5. 調整防火牆

   ```
   [root@bastion ~]# firewall-cmd --add-service=dns --permanent
   [root@bastion ~]# firewall-cmd --reload
   ```

6. 啟動服務

   ```
   [root@bastion ~]# systemctl enable named --now
   ```

7. 驗證DNS

   ```
   ##測試api
   [root@bastion ~]# dig +noall +answer @192.168.33.21 api.ocp.ken.lab
   api.ocp.ken.lab.	604800	IN	A	192.168.33.21
   
   [root@bastion ~]# nslookup 192.168.33.21
   21.33.168.192.in-addr.arpa	name = api.ocp.ken.lab.
   21.33.168.192.in-addr.arpa	name = bastion.ocp.ken.lab.
   21.33.168.192.in-addr.arpa	name = api-int.ocp.ken.lab.
   ## 測試 wild card
   [root@bastion ~]# dig +noall +answer @192.168.33.21 random.apps.ocp.ken.lab
   random.apps.ocp.ken.lab. 604800	IN	A	192.168.33.21
   ##測試反解析
   [root@bastion ~]# dig +noall +answer @192.168.33.21 -x 192.168.33.21
   21.33.168.192.in-addr.arpa. 10	IN	PTR	api.ocp.ken.lab.
   21.33.168.192.in-addr.arpa. 10	IN	PTR	api-int.ocp.ken.lab.
   21.33.168.192.in-addr.arpa. 10	IN	PTR	bastion.ocp.ken.lab.
   
   ```

---

### 安裝HA Proxy

1. 安裝haproxy

   ```
   [root@bastion ~]# yum install haproxy -y
   ```

2. 設定HAProxy

   ```
   將文件上的設定檔複製貼上，再用sed 指令修改domain name
   [root@bastion haproxy]# sed -i "s/ocp4.example.com/ocp.ken.lab/g" /etc/haproxy/haproxy.cfg
   [root@bastion ~]# cat /etc/haproxy/haproxy.cfg
   global
     log         127.0.0.1 local2
     pidfile     /var/run/haproxy.pid
     maxconn     4000
     daemon
   defaults
     mode                    http
     log                     global
     option                  dontlognull
     option http-server-close
     option                  redispatch
     retries                 3
     timeout http-request    10s
     timeout queue           1m
     timeout connect         10s
     timeout client          1m
     timeout server          1m
     timeout http-keep-alive 10s
     timeout check           10s
     maxconn                 3000
   frontend stats
     bind *:1936
     mode            http
     log             global
     maxconn 10
     stats enable
     stats hide-version
     stats refresh 30s
     stats show-node
     stats show-desc Stats for ocp4 cluster
     stats auth admin:ocp4
     stats uri /stats
   listen api-server-6443
     bind *:6443
     mode tcp
     server bootstrap bootstrap.ocp.ken.lab:6443 check inter 1s backup
     server master1 master1.ocp.ken.lab:6443 check inter 1s
     server master2 master3.ocp.ken.lab:6443 check inter 1s
     server master3 master3.ocp.ken.lab:6443 check inter 1s
   listen machine-config-server-22623
     bind *:22623
     mode tcp
     server bootstrap bootstrap.ocp.ken.lab:22623 check inter 1s backup 
     server master1 master1.ocp.ken.lab:22623 check inter 1s
     server master2 master2.ocp.ken.lab:22623 check inter 1s
     server master3 master3.ocp.ken.lab:22623 check inter 1s
   listen ingress-router-443
     bind *:443
     mode tcp
     balance source
     server worker1 worker1.ocp.ken.lab:443 check inter 1s
     server worker2 worker2.ocp.ken.lab:443 check inter 1s
     server infra1 infra1.ocp.ken.lab:443 check inter 1s
     server infra2 infra2.ocp.ken.lab:443 check inter 1s
     server infra3 infra3.ocp.ken.lab:443 check inter 1s
   listen ingress-router-80
     bind *:80
     mode tcp
     balance source
     server worker1 worker1.ocp.ken.lab:80 check inter 1s
     server worker2 worker2.ocp.ken.lab:80 check inter 1s
     server infra1 infra1.ocp.ken.lab:80 check inter 1s
     server infra2 infra2.ocp.ken.lab:80 check inter 1s
     server infra3 infra3.ocp.ken.lab:80 check inter 1s
   ```

3. 防火牆開通、SELinux設定

   ```
   [root@bastion ~]# firewall-cmd --add-port=80/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=443/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=6443/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=1936/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=22623/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=9000-9999/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=10250-10259/tcp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=4789/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=6081/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=9000-9999/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=500/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=4500/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=30000-32767/udp --permanent 
   success
   [root@bastion ~]# firewall-cmd --add-port=30000-32767/tcp --permanent 
   success
   
   [root@bastion ~]# firewall-cmd --reload
   success
   
   [root@bastion ~]# firewall-cmd --list-all
   public (active)
     target: default
     icmp-block-inversion: no
     interfaces: ens192
     sources: 
     services: cockpit dhcpv6-client dns ssh
     ports: 53/udp 53/tcp 80/tcp 443/tcp 6443/tcp 1936/tcp 22623/tcp 9000-9999/tcp 10250-10259/tcp 4789/udp 6081/udp 9000-9999/udp 500/udp 4500/udp 30000-32767/udp 30000-32767/tcp
     protocols: 
     forward: yes
     masquerade: no
     forward-ports: 
     source-ports: 
     icmp-blocks: 
     rich rules: 
   ## SELinux
   [root@bastion ocp]# semanage port --add --type http_port_t --proto tcp 1936
   [root@bastion ocp]# semanage port --add --type http_port_t --proto tcp 6443
   [root@bastion ocp]# semanage port --add --type http_port_t --proto tcp 22623
   or 
   [root@bastion ocp]# setsebool -P haproxy_connect_any=on
   ```

4. 啟動服務

   ```
   [root@bastion ~]# systemctl enable haproxy --now
   ```
   
---

### 下載Pull Secret Token

1. 網頁連線至https://console.redhat.com/openshift/downloads

2. 下載 Pull Secret Token
   ![image-20221011144551465](https://kenkenny.synology.me:5543/images/2023/09/image-20221011144551465.png)

3. 將 pull-secret放到安裝目錄下

   ```
   [devop@bastion ocp]$ cp ../pull-secret .
   [devop@bastion ocp]$ ll
   -rw-r--r--. 1 devop devop      2795 Oct 11 15:07 pull-secret
   ```

   ---

### 下載安裝openshift installer

1. 網頁連線至 https://console.redhat.com/openshift/downloads#tool-pull-secret

2. 下載 OpenShift for x86_64 Installer
   ![image-20221011145410398](https://kenkenny.synology.me:5543/images/2023/09/image-20221011145410398.png)

3. 創建目錄並解壓縮

   ```
   [devop@bastion ~]$ mkdir ocp
   [devop@bastion ~]$ mv openshift-install-linux.tar.gz ocp
   [devop@bastion ~]$ cd ocp
   [devop@bastion ocp]$ ll
   total 336508
   -rw-r--r--. 1 devop devop 344582118 Oct 11 14:59 openshift-install-linux.tar.gz
   [devop@bastion ocp]$ tar xzvf openshift-install-linux.tar.gz 
   README.md
   openshift-install
   
   ```

4. 將openshift installer 放到 /usr/local/bin

   ```
   [devop@bastion ocp]$ sudo cp openshift-install /usr/local/bin/
   [sudo] password for devop:
   [devop@bastion ocp]$ openshift-install --help
   Creates OpenShift clusters
   
   Usage:
     openshift-install [command]
     ..
   ```

---

### 下載 CoreOS

1. 連線至 https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.11/4.11.2/
2. 下載 https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.11/4.11.2/rhcos-4.11.2-x86_64-live.x86_64.iso
3. 放到ESXi主機上並掛給各虛擬機
   ![image-20221011150628575](https://kenkenny.synology.me:5543/images/2023/09/image-20221011150628575.png)
4. ![image-20221011151238082](https://kenkenny.synology.me:5543/images/2023/09/image-20221011151238082.png)

---

### 準備安裝Soruce

#### 創建install-config.yaml

在/home/devop/ocp目錄下 新增install-config.yaml
重裝時記得將此目錄清除乾淨

```
[devop@bastion ocp]$ vim install-config.yaml 
[devop@bastion ocp]$ cat install-config.yaml 
apiVersion: v1
baseDomain: ken.lab 
compute: 
- hyperthreading: Enabled 
  name: worker
  replicas: 0 
controlPlane: 
  hyperthreading: Enabled 
  name: master
  replicas: 3 
metadata:
  name: ocp
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"}}}'
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIN1Z2jXmGnPWGAVrtltN9A2q5D8QtJlRvhvdB2QXoZG devop@bastion.ocp.ken.lab' 
capabilities:
  baselineCapabilitySet: vCurrent


-------### 只裝Operator Hub ###---------
capabilities:
  baselineCapabilitySet: None
  additionalEnabledCapabilities:
  - marketplace
```

#### 建立K8S manifest  files

在相同目錄下執行
重裝時記得將此目錄清除乾淨

```
[devop@bastion ocp]$ openshift-install create manifests --dir /home/devop/ocp/install/
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 
INFO Manifests created in: /home/devop/ocp/install/manifests and /home/devop/ocp/install/openshift
```

#### ＭastersSchedulable:false

```
[devop@bastion ocp]$ cd manifests/
[devop@bastion manifests]$ cat cluster-scheduler-02-config.yml 
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: null
  name: cluster
spec:
  mastersSchedulable: false
  policy:
    name: ""
status: {}
```

#### 創建Ignition config file

重裝時記得將此目錄清除乾淨

```
[devop@bastion ocp]$ openshift-install create ignition-configs --dir /home/devop/ocp/install/
INFO Consuming Worker Machines from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Consuming Master Machines from target directory 
INFO Ignition-Configs created in: /home/devop/ocp/install and /home/devop/ocp/install/auth 
---
[devop@bastion ocp]$ tree install/
install/
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── worker.ign

1 directory, 6 files


##確認 sha512sum
[root@bastion ocp]# sha512sum bootstrap.ign 
460f76f9bc8b1df7095326c6b581b60f1150e477ae416b065c7b18ae040dc530a3086857d649ae04f646ccf9cce16ea9a4c32c8fe48b3630413f8ebe3f924329  bootstrap.ign
```

### 安裝Http Web Server

1. 安裝httpd

   ```
   [root@bastion ~]# yum install httpd -y
   ```

2. 設定httpd 預設port 8989、修改SELinux

   ```
   [root@bastion ~]# vim /etc/httpd/conf/httpd.conf 
   
   Listen 8989
   
   [root@bastion ocp]# firewall-cmd --add-port=8989/tcp --permanent 
   success
   [root@bastion ocp]# firewall-cmd --reload 
   success
   [root@bastion ocp]# semanage port -a -t http_port_t -p tcp 8989
   ```

3. 將ignition檔案複製到 /var/www/html

   ```
   [root@bastion ~]# cp /home/devop/ocp/*.ign /var/www/html/ocp/
   [root@bastion ~]# cp /var/www/html/ocp
   [root@bastion ocp]# chmod 644 *.ign
   ```

4. 啟動httpd並測試連線

   ```
   [root@bastion ocp]# systemctl enable httpd --now
   [root@bastion ocp]# curl http://192.168.33.21:8989/ocp/bootstrap.ign
   ```

---

## 安裝BootStrap

1. 將虛擬機開機，並使用rhel core os 的iso檔開機，
   若前面dhcp server、dns server設定成功，即會自動拿到FQDN、IP
   ![image-20221011155843248](https://kenkenny.synology.me:5543/images/2023/09/image-20221011155843248.png)

2. 在bootstrap執行安裝指令

   ```
   sudo coreos-installer install --ignition-url=http://192.168.33.21:8989/ocp/bootstrap.ign /dev/sda --insecure-ignition
   (--insecure-ignition can replace using --ignition-hash=sha512-xxxxxxxxxxxxx)
   ```

3. 出現此畫面後 執行重新啟動 reboot
   ![image-20221011165057146](https://kenkenny.synology.me:5543/images/2023/09/image-20221011165057146.png)

4. 開機後確認已寫入硬碟
   ![image-20221011165808592](https://kenkenny.synology.me:5543/images/2023/09/image-20221011165808592.png)

5. 在bastion登入bootstrap

   ```
   [devop@bastion ~]$ ssh core@bootstrap
   The authenticity of host 'bootstrap (192.168.33.22)' can't be established.
   ED25519 key fingerprint is SHA256:pnQoMRtW8f/3EjIQ7qNG4+NIB+ObrTd4E8aEbvRqMQw.
   This key is not known by any other names
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   Warning: Permanently added 'bootstrap' (ED25519) to the list of known hosts.
   client_global_hostkeys_private_confirm: server gave bad signature for RSA key 0: error in libcrypto
   Red Hat Enterprise Linux CoreOS 411.86.202208112011-0
     Part of OpenShift 4.11, RHCOS is a Kubernetes native operating system
     managed by the Machine Config Operator (`clusteroperator/machine-config`).
   
   WARNING: Direct SSH access to machines is not recommended; instead,
   make configuration changes via `machineconfig` objects:
     https://docs.openshift.com/container-platform/4.11/architecture/architecture-rhcos.html
   
   ---
   This is the bootstrap node; it will be destroyed when the master is fully up.
   
   The primary services are release-image.service followed by bootkube.service. To watch their status, run e.g.
   
     journalctl -b -f -u release-image.service -u bootkube.service
   
   ```

6. 查看log，到出現machine api時才可接續安裝

   ```
   [core@bootstrap ~]$ journalctl -b -f -u release-image.service -u bootkube.service
   Create "99_openshift-cluster-api_worker-user-data-secret.yaml" secrets.v1./worker-user-data -n openshift-machine-api
   
   ```

7. 此時才可繼續安裝 master01、master02、master03
   ![image-20221011173412330](https://kenkenny.synology.me:5543/images/2023/09/image-20221011173412330.png)

8. 出現此畫面即可關閉bootstrap![image-20221011172057819](https://kenkenny.synology.me:5543/images/2023/09/image-20221011172057819.png)

   ![image-20221013110951260](https://kenkenny.synology.me:5543/images/2023/09/image-20221013110951260.png)

## 安裝 Ｍaster Node

1. 將虛擬機開機，並使用rhel core os 的iso檔開機，
   若前面dhcp server、dns server設定成功，即會自動拿到FQDN、IP

2. 在master執行安裝指令，三台相同 (若設定ＭastersSchedulable:false，則worker需一起安裝）

   ```
   sudo coreos-installer install --ignition-url=http://192.168.33.21:8989/ocp/master.ign /dev/sda --insecure-ignition
   出現install complete 即可reboot
   ```

   ![image-20221012012023876](https://kenkenny.synology.me:5543/images/2023/09/image-20221012012023876.png)

3. 持續監控bootstrap的 Log

   ```
   [core@bootstrap ~]$ journalctl -b -f -u release-image.service -u bootkube.service
   ```

4. Master Node裝完會自動重開機

5. 在bastion 驗證節點

   ```
   [devop@bastion ~]$ oc get nodes
   NAME                  STATUS   ROLES    AGE     VERSION
   master1.ocp.ken.lab   Ready    master   4m22s   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master   3m37s   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master   3m21s   v1.24.0+3882f8f
   ```
   
6. 查看bootstrap的 Log,確認安裝完成

   ![image-20221012013436842](https://kenkenny.synology.me:5543/images/2023/09/image-20221012013436842.png)

   ![image-20221012014537780](https://kenkenny.synology.me:5543/images/2023/09/image-20221012014537780.png)

7. 在Bastion 執行oc 指令確認ocp cluster CSR

   ```
   [devop@bastion ~]$ oc get nodes
   NAME                  STATUS   ROLES    AGE   VERSION
   master1.ocp.ken.lab   Ready    master   18m   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master   17m   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master   17m   v1.24.0+3882f8f
   
   [devop@bastion ~]$ oc get csr
   NAME                                             AGE   SIGNERNAME                                    REQUESTOR                                                                         REQUESTEDDURATION   CONDITION
   csr-7sgfx                                        18m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-lphdj                                        17m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-ndft5                                        17m   kubernetes.io/kubelet-serving                 system:node:master3.ocp.ken.lab                                                   <none>              Approved,Issued
   csr-p42p5                                        17m   kubernetes.io/kubelet-serving                 system:node:master2.ocp.ken.lab                                                   <none>              Approved,Issued
   csr-vgmd9                                        17m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-zpfw4                                        18m   kubernetes.io/kubelet-serving                 system:node:master1.ocp.ken.lab                                                   <none>              Approved,Issued
   system:openshift:openshift-authenticator-2zxkf   15m   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-authentication-operator:authentication-operator   <none>              Approved,Issued
   system:openshift:openshift-monitoring-ln69x      15m   kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-monitoring:cluster-monitoring-operator            <none>              Approved,Issued
   ```

8. 確認cluster operator

   ```
   [devop@bastion install]$ oc get co -A
   NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
   authentication                             4.11.7    False       False         False      76s     APIServicesAvailable: PreconditionNotReady...
   cloud-controller-manager                   4.11.7    True        False         False      2m36s   
   cloud-credential                                     True        False         False      13m     
   cluster-autoscaler                                                                                
   config-operator                            4.11.7    True        False         False      40s     
   console                                                                                           
   csi-snapshot-controller                    4.11.7    True        False         False      1s      
   dns                                                                                               
   etcd                                                                                              
   image-registry                                                                                    
   ingress                                                                                           
   insights                                                                                          
   kube-apiserver                                       Unknown     False         False      112s    
   kube-controller-manager                              False       True          False      75s     StaticPodsAvailable: 0 nodes are active; 3 nodes are at revision 0; 0 nodes have achieved new revision 2
   kube-scheduler                                       False       True          False      35s     StaticPodsAvailable: 0 nodes are active; 3 nodes are at revision 0; 0 nodes have achieved new revision 3
   kube-storage-version-migrator              4.11.7    True        False         False      59s     
   machine-api                                                                                       
   machine-approver                                                                                  
   machine-config                                                   True                             Working towards 4.11.7
   marketplace                                                                                       
   monitoring                                                                                        
   network                                              False       True          False      2m53s   The network is starting up
   node-tuning                                                                                       
   openshift-apiserver                                                                               
   openshift-controller-manager                         False       True          False      100s    Available: no daemon pods available on any node.
   openshift-samples                                                                                 
   operator-lifecycle-manager                                                                        
   operator-lifecycle-manager-catalog                                                                
   operator-lifecycle-manager-packageserver                                                          
   service-ca                                 4.11.7    True        False         False      33s     
   storage                                    4.11.7    True        False         False      49s   
   ```

9. 關閉bootstrap，安裝worker node

## 安裝 Worker＆Infra Node

https://docs.openshift.com/container-platform/4.11/machine_management/user_infra/adding-bare-metal-compute-user-infra.html

1. 將虛擬機開機，並使用rhel core os 的iso檔開機，
   若前面dhcp server、dns server設定成功，即會自動拿到FQDN、IP

2. 執行安裝指令 (worker1、worker2、infra1、infra2、infra3)

   ```
   sudo coreos-installer install --ignition-url=http://192.168.33.21:8989/ocp/worker.ign /dev/sda --insecure-ignition
   出現install complete 即可reboot
   ```

3. 在bastion 確認nodes、csr (會有很多需approve的 csr，後續加node後還會有)

   ```
   [devop@bastion ~]$ oc get nodes
   NAME                  STATUS   ROLES    AGE   VERSION
   master1.ocp.ken.lab   Ready    master   36m   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master   35m   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master   35m   v1.24.0+3882f8f
   ----
   [devop@bastion ~]$ oc get csr
   NAME                                             AGE    SIGNERNAME                                    REQUESTOR                                                                         REQUESTEDDURATION   CONDITION
   csr-7sgfx                                        37m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-lphdj                                        37m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-ndft5                                        36m    kubernetes.io/kubelet-serving                 system:node:master3.ocp.ken.lab                                                   <none>              Approved,Issued
   csr-nwplh                                        0s     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Pending
   csr-p42p5                                        36m    kubernetes.io/kubelet-serving                 system:node:master2.ocp.ken.lab                                                   <none>              Approved,Issued
   csr-pcsl2                                        115s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Pending
   csr-vgmd9                                        36m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Approved,Issued
   csr-zpfw4                                        37m    kubernetes.io/kubelet-serving                 system:node:master1.ocp.ken.lab                                                   <none>              Approved,Issued
   system:openshift:openshift-authenticator-2zxkf   35m    kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-authentication-operator:authentication-operator   <none>              Approved,Issued
   system:openshift:openshift-monitoring-ln69x      34m    kubernetes.io/kube-apiserver-client           system:serviceaccount:openshift-monitoring:cluster-monitoring-operator            <none>              Approved,Issued
   
   ```

4. Approve All CSR

   ```
   $oc adm certificate approve <csr_name>
   ----
   $oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
   ----
   [devop@bastion auth]$ oc get csr -A
   NAME        AGE     SIGNERNAME                                    REQUESTOR                                                                   REQUESTEDDURATION   CONDITION
   csr-2trmm   18m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-9xhq4   16m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-b4jns   2m45s   kubernetes.io/kubelet-serving                 system:node:infra1.ocp.ken.lab                                              <none>              Pending
   csr-b55jd   3m57s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-bdtnw   16m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-bngbr   2m40s   kubernetes.io/kubelet-serving                 system:node:infra2.ocp.ken.lab                                              <none>              Pending
   csr-kd66h   2m51s   kubernetes.io/kubelet-serving                 system:node:infra2.ocp.ken.lab                                              <none>              Pending
   csr-lzk6w   2m46s   kubernetes.io/kubelet-serving                 system:node:worker1.ocp.ken.lab                                             <none>              Pending
   csr-rtkr6   2m46s   kubernetes.io/kubelet-serving                 system:node:worker2.ocp.ken.lab                                             <none>              Pending
   csr-swwhm   18m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-v67v6   16m     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   csr-vhlqb   3m56s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
   ---
   [devop@bastion auth]$ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
   certificatesigningrequest.certificates.k8s.io/csr-2trmm approved
   certificatesigningrequest.certificates.k8s.io/csr-9xhq4 approved
   certificatesigningrequest.certificates.k8s.io/csr-b55jd approved
   certificatesigningrequest.certificates.k8s.io/csr-bdtnw approved
   certificatesigningrequest.certificates.k8s.io/csr-swwhm approved
   certificatesigningrequest.certificates.k8s.io/csr-v67v6 approved
   certificatesigningrequest.certificates.k8s.io/csr-vhlqb approved
   ---
   [devop@bastion auth]$ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
   certificatesigningrequest.certificates.k8s.io/csr-b4jns approved
   certificatesigningrequest.certificates.k8s.io/csr-bngbr approved
   certificatesigningrequest.certificates.k8s.io/csr-cq46l approved
   certificatesigningrequest.certificates.k8s.io/csr-kd66h approved
   certificatesigningrequest.certificates.k8s.io/csr-lzk6w approved
   certificatesigningrequest.certificates.k8s.io/csr-rtkr6 approved
   certificatesigningrequest.certificates.k8s.io/csr-t6l2b approved
   
   ```

5. 再次確認node狀態

   ```
   [devop@bastion auth]$ oc get nodes
   NAME                  STATUS                     ROLES    AGE     VERSION
   infra1.ocp.ken.lab    Ready                      worker   2m32s   v1.24.0+3882f8f
   infra2.ocp.ken.lab    Ready,SchedulingDisabled   worker   2m38s   v1.24.0+3882f8f
   master1.ocp.ken.lab   NotReady                   master   78m     v1.24.0+3882f8f
   master2.ocp.ken.lab   NotReady                   master   78m     v1.24.0+3882f8f
   master3.ocp.ken.lab   NotReady                   master   78m     v1.24.0+3882f8f
   worker1.ocp.ken.lab   Ready                      worker   2m33s   v1.24.0+3882f8f
   worker2.ocp.ken.lab   Ready                      worker   2m34s   v1.24.0+3882f8f
   
   ---
   [devop@bastion ~]$ oc get nodes
   NAME                  STATUS   ROLES    AGE   VERSION
   master1.ocp.ken.lab   Ready    master   39m   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master   39m   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master   38m   v1.24.0+3882f8f
   worker1.ocp.ken.lab   Ready    worker   95s   v1.24.0+3882f8f
   worker2.ocp.ken.lab   Ready    worker   90s   v1.24.0+3882f8f
   ---
   [devop@bastion ~]$ oc get po -A
   ...
   ```

## 安裝後的設定

#### 設定NFS

https://access.redhat.com/solutions/4618011

https://access.redhat.com/solutions/5900501

on vmware vsphere

https://vergiehadiana.medium.com/guide-deploy-nfs-storage-for-dynamic-provisioning-and-image-registry-for-red-hat-openshift-b15e16f357b4

https://hackmd.io/@kd-ocp/S1EIu6V25

```
image:
k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2

image:
groundhog2k/nfs-subdir-external-provisioner:v3.2.0
```

#### 設定外部Image Registry

https://access.redhat.com/documentation/zh-cn/openshift_container_platform/4.6/html/images/images-configuration-file_image-configuration

#### 設定Htpasswd Provider

https://docs.openshift.com/container-platform/4.11/authentication/identity_providers/configuring-htpasswd-identity-provider.html

1. 利用htpasswd 產生htpasswd檔案

   ```
   [devop@bastion ocp]$ htpasswd -c -B -b users.htpasswd admin redhat
   Adding password for user admin
   [devop@bastion ocp]$ htpasswd -B -b users.htpasswd ken xxxxxxxx
   Adding password for user ken
   [devop@bastion ocp]$ htpasswd -B -b users.htpasswd view redhat
   Adding password for user view
   ```

2. 產生secret 將 htpasswd 放入secret

   ```
   [devop@bastion ocp]$ oc whoami
   system:admin
   [devop@bastion ocp]$ oc create secret generic users-htpasswd --from-file htpasswd=users.htpasswd -n openshift-config
   secret/users-htpasswd created
   [devop@bastion ocp]$ oc get secret -n openshift-config
   NAME                                      TYPE                                  DATA   AGE
   builder-dockercfg-r46z8                   kubernetes.io/dockercfg               1      8h
   builder-token-cgvp8                       kubernetes.io/service-account-token   4      8h
   default-dockercfg-8mhbj                   kubernetes.io/dockercfg               1      8h
   default-token-nfc27                       kubernetes.io/service-account-token   4      8h
   deployer-dockercfg-qdwcq                  kubernetes.io/dockercfg               1      8h
   deployer-token-bfplj                      kubernetes.io/service-account-token   4      8h
   etcd-client                               kubernetes.io/tls                     2      8h
   etcd-metric-client                        kubernetes.io/tls                     2      8h
   etcd-metric-signer                        kubernetes.io/tls                     2      8h
   etcd-signer                               kubernetes.io/tls                     2      8h
   initial-service-account-private-key       Opaque                                1      8h
   pull-secret                               kubernetes.io/dockerconfigjson        1      8h
   users-htpasswd                            Opaque                                1      10s
   webhook-authentication-integrated-oauth   Opaque                                1      8h
   ```

3. 新增 htpasswd provider

   ```
   [devop@bastion ocp]$ oc get oauth cluster -o yaml > oauth.yaml
   [devop@bastion ocp]$ vim oauth.yaml 
   [devop@bastion ocp]$ cat oauth.yaml 
   apiVersion: config.openshift.io/v1
   kind: OAuth
   metadata:
     annotations:
       include.release.openshift.io/ibm-cloud-managed: "true"
       include.release.openshift.io/self-managed-high-availability: "true"
       include.release.openshift.io/single-node-developer: "true"
       release.openshift.io/create-only: "true"
     creationTimestamp: "2022-10-11T17:14:37Z"
     generation: 1
     name: cluster
     ownerReferences:
     - apiVersion: config.openshift.io/v1
       kind: ClusterVersion
       name: version
       uid: d1c0dbf6-000f-4b1e-aa27-1bd8a667afac
     resourceVersion: "1533"
     uid: 3e95cfe7-7c02-47eb-8b4e-7f0d61766461
   spec:
     identityProviders:
     - name: htpasswd_provider 
       mappingMethod: claim 
       type: HTPasswd
       htpasswd:
         fileData:
           name: users-htpasswd
   
   [devop@bastion ocp]$ oc apply -f oauth.yaml -n openshift-config
   Warning: resource oauths/cluster is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
   oauth.config.openshift.io/cluster configured
   ---
   [devop@bastion ocp]$ watch oc get po -n openshift-authentication
   ```

   ![image-20221012094929634](https://kenkenny.synology.me:5543/images/2023/09/image-20221012094929634.png)

4. 調整角色權限 

   ```
   ## 將cluster-admin role 加給 admin、ken
   [devop@bastion ~]$ oc adm policy add-cluster-role-to-user cluster-admin admin
   clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"
   [devop@bastion ~]$ oc adm policy add-cluster-role-to-user cluster-admin ken
   Warning: User 'ken' not found
   clusterrole.rbac.authorization.k8s.io/cluster-admin added: "ken"
   
   ## 將cluster-view role 加給 view
   [devop@bastion ~]$ oc get clusterroles | grep view
   view                                                            2022-10-11T17:13:19Z
   ...
   [devop@bastion ~]$ oc adm policy add-cluster-role-to-user view view
   Warning: User 'view' not found
   clusterrole.rbac.authorization.k8s.io/view added: "view"
   ```

5. 將預設可建立project的權限刪除，只給admin、ken

   ```
   ## 取得 self-provisioner的role
   [devop@bastion ~]$ oc get clusterrolebindings | grep self-provisioner
   self-provisioners                                                           ClusterRole/self-provisioner   8h
   
   ## 取得 self-provisioners role binding的group
   [devop@bastion ~]$ oc describe clusterrolebindings self-provisioners
   Name:         self-provisioners
   Labels:       <none>
   Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
   Role:
     Kind:  ClusterRole
     Name:  self-provisioner
   Subjects:
     Kind   Name                        Namespace
     ----   ----                        ---------
     Group  system:authenticated:oauth  
     
   ##移除 self-provisioner的rolebinding
   [devop@bastion ~]$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
   Warning: Your changes may get lost whenever a master is restarted, unless you prevent reconciliation of this rolebinding using the following command: oc annotate clusterrolebinding.rbac self-provisioners 'rbac.authorization.kubernetes.io/autoupdate=false' --overwrite
   clusterrole.rbac.authorization.k8s.io/self-provisioner removed: "system:authenticated:oauth"
   
   ##新增可以建立project的group
   [devop@bastion ~]$ oc adm groups new new-project-group admin ken
   group.user.openshift.io/new-project-group created
   
   ##將new-project-group再bind上self-provisioner
   [devop@bastion ~]$ oc adm policy add-cluster-role-to-group self-provisioner new-project-group
   clusterrole.rbac.authorization.k8s.io/self-provisioner added: "new-project-group"
   ```

6. 取得console url 

   ```
   [devop@bastion ocp]$ oc get route -n openshift-console
   NAME        HOST/PORT                                      PATH   SERVICES    PORT    TERMINATION          WILDCARD
   console     console-openshift-console.apps.ocp.ken.lab            console     https   reencrypt/Redirect   None
   downloads   downloads-openshift-console.apps.ocp.ken.lab          downloads   http    edge/Redirect        None
   ### 刪除 kubeadmin
   [devop@bastion ocp]$ oc delete secret kubeadmin -n kube-system
   ```

7. 連線至console  https://console-openshift-console.apps.ocp.ken.lab 並測試登入

   admin / redhat (cluster-admin)、view / redhat (view only)
   ![image-20221012095225994](https://kenkenny.synology.me:5543/images/2023/09/image-20221012095225994.png)

   ![image-20221012095300179](https://kenkenny.synology.me:5543/images/2023/09/image-20221012095300179.png)
   ![image-20221012095339064](https://kenkenny.synology.me:5543/images/2023/09/image-20221012095339064.png)

#### 註冊openshift cluster

1. 從console 點到 Administration > Cluster Settings > Manage subscription settings
   ![image-20221012101701215](https://kenkenny.synology.me:5543/images/2023/09/image-20221012101701215.png)

2. 點選Edit subscription settings

   ![image-20221012101731793](https://kenkenny.synology.me:5543/images/2023/09/image-20221012101731793.png)

3. 選擇 Self-Support 即可
   ![image-20221012101910450](https://kenkenny.synology.me:5543/images/2023/09/image-20221012101910450.png)

4. 約 3~5 分鐘即可更新License

   ![image-20221012102253203](https://kenkenny.synology.me:5543/images/2023/09/image-20221012102253203.png)12

#### 設定OC CLI CA

在openshift-authentication project內取得CA並將CA複製到bastion內

```
[devop@bastion ocp]$ oc project openshift-authentication
Now using project "openshift-authentication" on server "https://api.ocp.ken.lab:6443".
[devop@bastion ocp]$ oc get po
NAME                               READY   STATUS    RESTARTS   AGE
oauth-openshift-7885f5bd5b-4m5sh   1/1     Running   0          46m
oauth-openshift-7885f5bd5b-q6vbk   1/1     Running   0          46m
oauth-openshift-7885f5bd5b-vh4pz   1/1     Running   0          47m
[devop@bastion ocp]$ oc rsh oauth-openshift-7885f5bd5b-4m5sh cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ca.crt
＃＃切換root
[root@bastion ocp]# cp /home/devop/ocp/ca.crt /etc/pki/ca-trust/source/anchors/
[root@bastion ocp]# update-ca-trust extract
## 切換為devop測試登入
[devop@bastion ~]$ oc login -u admin -p redhat https://api.ocp.ken.lab:6443
Login successful.

You have access to 66 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "openshift-authentication".
```

#### 拆分Infra Node

https://docs.openshift.com/container-platform/4.11/post_installation_configuration/cluster-tasks.html#post-install-cluster-tasks

https://access.redhat.com/solutions/4601131

1. 針對infra node 建立label 

   ```
   [devop@bastion ocp]$ oc label node infra1.ocp.ken.lab node-role.kubernetes.io/infra=""
   node/infra1.ocp.ken.lab labeled
   [devop@bastion ocp]$ oc label node infra2.ocp.ken.lab node-role.kubernetes.io/infra=""
   node/infra2.ocp.ken.lab labeled
   [devop@bastion ocp]$ oc label node infra3.ocp.ken.lab node-role.kubernetes.io/infra=""
   node/infra3.ocp.ken.lab labeled
   [devop@bastion ocp]$ oc get nodes
   NAME                  STATUS   ROLES          AGE     VERSION
   infra1.ocp.ken.lab    Ready    infra,worker   4h1m    v1.24.0+3882f8f
   infra2.ocp.ken.lab    Ready    infra,worker   4h1m    v1.24.0+3882f8f
   infra3.ocp.ken.lab    Ready    infra,worker   3h19m   v1.24.0+3882f8f
   master1.ocp.ken.lab   Ready    master         4h43m   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master         4h42m   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master         4h41m   v1.24.0+3882f8f
   worker1.ocp.ken.lab   Ready    worker         4h17m   v1.24.0+3882f8f
   worker2.ocp.ken.lab   Ready    worker         4h17m   v1.24.0+3882f8f
   ```

2. 建立machine config pool  >> infra.mcp.yaml 並apply到 cluster

   ```
   [devop@bastion ocp]$ vim infra.mcp.yaml
   
   apiVersion: machineconfiguration.openshift.io/v1
   kind: MachineConfigPool
   metadata:
     name: infra
   spec:
     machineConfigSelector:
       matchExpressions:
         - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,infra]} 
     nodeSelector:
       matchLabels:
         node-role.kubernetes.io/infra: ""
         
   [devop@bastion ocp]$ oc apply -f infra.mcp.yaml 
   machineconfigpool.machineconfiguration.openshift.io/infra created
   
   [devop@bastion ocp]$ oc get  machineconfig
   NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
   00-master                                          b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   00-worker                                          b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   01-master-container-runtime                        b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   01-master-kubelet                                  b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   01-worker-container-runtime                        b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   01-worker-kubelet                                  b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   99-master-generated-registries                     b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   99-master-ssh                                                                                 3.2.0             5h29m
   99-worker-generated-registries                     b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   99-worker-ssh                                                                                 3.2.0             5h29m
   rendered-infra-407eaba9df9f726f52f84b938fce9025    b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             3s
   rendered-master-720c11fac56674c6fc876ab6161295aa   b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   rendered-worker-407eaba9df9f726f52f84b938fce9025   b7aa1d499f15ce7d211d30a823acf83f47aabbe3   3.2.0             5h12m
   
   ```

3. 建立 machine config >> infra.mc.yaml 並apply到cluster

   ```
   [devop@bastion ocp]$ vim infra.mc.yaml
   apiVersion: machineconfiguration.openshift.io/v1
   kind: MachineConfig
   metadata:
     name: 51-infra
     labels:
       machineconfiguration.openshift.io/role: infra 
   spec:
     config:
       ignition:
         version: 3.2.0
       storage:
         files:
         - path: /etc/infratest
           mode: 0644
           contents:
             source: data:,infra
        
   [devop@bastion ocp]$ oc create -f infra.mc.yaml
   machineconfig.machineconfiguration.openshift.io/51-infra created
   [devop@bastion ocp]$ oc get mcp
   NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
   infra    rendered-infra-407eaba9df9f726f52f84b938fce9025    False     True       False      3              0                   0                     0                      3m17s
   master   rendered-master-720c11fac56674c6fc876ab6161295aa   True      False      False      3              3                   3                     0                      5h25m
   worker   rendered-worker-407eaba9df9f726f52f84b938fce9025   True      False      False      2              2                   2                     0                      5h25m
   ```

4. Add a Taint To Infra Node ，使Infra Node不會被app部署到
   #此動作可搬完operator後再做

   ```
   oc adm taint nodes node1 node-role.kubernetes.io/infra:NoSchedule
   [devop@bastion ocp]$ oc adm taint nodes infra3.ocp.ken.lab node-role.kubernetes.io/infra:NoSchedule
   node/infra3.ocp.ken.lab tainted
   [devop@bastion ocp]$ oc adm taint nodes infra2.ocp.ken.lab node-role.kubernetes.io/infra:NoSchedule
   node/infra2.ocp.ken.lab tainted
   [devop@bastion ocp]$ oc adm taint nodes infra1.ocp.ken.lab node-role.kubernetes.io/infra:NoSchedule
   node/infra1.ocp.ken.lab tainted
   
   [devop@bastion ocp]$ oc describe nodes infra1
   Name:               infra1.ocp.ken.lab
   Roles:              infra,worker
   Labels:             beta.kubernetes.io/arch=amd64
                       beta.kubernetes.io/os=linux
                       kubernetes.io/arch=amd64
                       kubernetes.io/hostname=infra1.ocp.ken.lab
                       kubernetes.io/os=linux
                       node-role.kubernetes.io/infra=
                       node-role.kubernetes.io/worker=
                       node.openshift.io/os_id=rhcos
   Annotations:        machineconfiguration.openshift.io/controlPlaneTopology: HighlyAvailable
                       machineconfiguration.openshift.io/currentConfig: rendered-infra-c36859c83539d92df71208a425c004df
                       machineconfiguration.openshift.io/desiredConfig: rendered-infra-c36859c83539d92df71208a425c004df
                       machineconfiguration.openshift.io/desiredDrain: uncordon-rendered-infra-c36859c83539d92df71208a425c004df
                       machineconfiguration.openshift.io/lastAppliedDrain: uncordon-rendered-infra-c36859c83539d92df71208a425c004df
                       machineconfiguration.openshift.io/reason: 
                       machineconfiguration.openshift.io/state: Done
                       volumes.kubernetes.io/controller-managed-attach-detach: true
   CreationTimestamp:  Thu, 13 Oct 2022 11:26:50 +0800
   Taints:             node-role.kubernetes.io/infra:NoSchedule
   Unschedulable:      false
   Lease:
     HolderIdentity:  infra1.ocp.ken.lab
     AcquireTime:     <unset>
     RenewTime:       Thu, 13 Oct 2022 16:23:54 +0800
   ```

5. 刪除infra node的 worker label

   ```
   [devop@bastion ocp]$ oc edit nodes infra1.ocp.ken.lab
   node/infra1.ocp.ken.lab edited
    labels:
       beta.kubernetes.io/arch: amd64
       beta.kubernetes.io/os: linux
       kubernetes.io/arch: amd64
       kubernetes.io/hostname: infra2.ocp.ken.lab
       kubernetes.io/os: linux
       node-role.kubernetes.io/infra: ""
       node-role.kubernetes.io/worker: ""  >>>> 刪除此行
       node.openshift.io/os_id: rhcos
   [devop@bastion ocp]$ oc get nodes
   NAME                  STATUS   ROLES    AGE   VERSION
   infra1.ocp.ken.lab    Ready    infra    22h   v1.24.0+3882f8f
   infra2.ocp.ken.lab    Ready    infra    22h   v1.24.0+3882f8f
   infra3.ocp.ken.lab    Ready    infra    22h   v1.24.0+3882f8f
   master1.ocp.ken.lab   Ready    master   23h   v1.24.0+3882f8f
   master2.ocp.ken.lab   Ready    master   23h   v1.24.0+3882f8f
   master3.ocp.ken.lab   Ready    master   23h   v1.24.0+3882f8f
   worker1.ocp.ken.lab   Ready    worker   22h   v1.24.0+3882f8f
   worker2.ocp.ken.lab   Ready    worker   22h   v1.24.0+3882f8f
   ```

#### Operator移動到Infra Node

##### Router

```
方法一
[devop@bastion ocp]$ oc get ingresscontroller default -n openshift-ingress-operator -o yaml > router.yaml
[devop@bastion ocp]$ vim router.yaml 
[devop@bastion ocp]$ cat router.yaml 
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  creationTimestamp: "2022-10-13T02:54:05Z"
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  generation: 1
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "26400"
  uid: 1d5c3f18-0f53-4140-be74-35df5edf4c8b
spec:
  clientTLS:
    clientCA:
      name: ""
    clientCertificatePolicy: ""
  httpCompression: {}
  httpEmptyRequestsPolicy: Respond
  httpErrorCodePages:
    name: ""
  replicas: 2
  tuningOptions: {}
  unsupportedConfigOverrides: null
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2022-10-13T02:54:13Z"
    reason: Valid
    status: "True"
    type: Admitted
  - lastTransitionTime: "2022-10-13T03:12:03Z"
    status: "True"
    type: PodsScheduled
  - lastTransitionTime: "2022-10-13T03:11:54Z"
    message: The deployment has Available status condition set to True
    reason: DeploymentAvailable
    status: "True"
    type: DeploymentAvailable
  - lastTransitionTime: "2022-10-13T03:11:54Z"
    message: Minimum replicas requirement is met
    reason: DeploymentMinimumReplicasMet
    status: "True"
    type: DeploymentReplicasMinAvailable
  - lastTransitionTime: "2022-10-13T03:12:36Z"
    message: All replicas are available
    reason: DeploymentReplicasAvailable
    status: "True"
    type: DeploymentReplicasAllAvailable
  - lastTransitionTime: "2022-10-13T02:54:13Z"
    message: The configured endpoint publishing strategy does not include a managed
      load balancer
    reason: EndpointPublishingStrategyExcludesManagedLoadBalancer
    status: "False"
    type: LoadBalancerManaged
  - lastTransitionTime: "2022-10-13T02:54:13Z"
    message: No DNS zones are defined in the cluster dns config.
    reason: NoDNSZones
    status: "False"
    type: DNSManaged
  - lastTransitionTime: "2022-10-13T03:11:54Z"
    status: "True"
    type: Available
  - lastTransitionTime: "2022-10-13T02:54:13Z"
    status: "False"
    type: Progressing
  - lastTransitionTime: "2022-10-13T03:12:03Z"
    status: "False"
    type: Degraded
  - lastTransitionTime: "2022-10-13T02:54:13Z"
    message: IngressController is upgradeable.
    reason: Upgradeable
    status: "True"
    type: Upgradeable
  - lastTransitionTime: "2022-10-13T03:12:02Z"
    message: Canary route checks for the default ingress controller are successful
    reason: CanaryChecksSucceeding
    status: "True"
    type: CanaryChecksSucceeding
  domain: apps.ocp.ken.lab
  endpointPublishingStrategy:
    hostNetwork:
      httpPort: 80
      httpsPort: 443
      protocol: TCP
      statsPort: 1936
    type: HostNetwork
  observedGeneration: 1
  selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
  tlsProfile:
    ciphers:
    - ECDHE-ECDSA-AES128-GCM-SHA256
    - ECDHE-RSA-AES128-GCM-SHA256
    - ECDHE-ECDSA-AES256-GCM-SHA384
    - ECDHE-RSA-AES256-GCM-SHA384
    - ECDHE-ECDSA-CHACHA20-POLY1305
    - ECDHE-RSA-CHACHA20-POLY1305
    - DHE-RSA-AES128-GCM-SHA256
    - DHE-RSA-AES256-GCM-SHA384
    - TLS_AES_128_GCM_SHA256
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
    minTLSVersion: VersionTLS12
[devop@bastion ocp]$ oc apply -f router.yaml -n openshift-ingress-operator
Warning: resource ingresscontrollers/default is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
ingresscontroller.operator.openshift.io/default configured

方法二 （官方建議）
[devop@bastion ocp]$ oc edit ingresscontroller -n openshift-ingress-operator
####加入下方 matchLabels
  spec:
    nodePlacement:
      nodeSelector:
        matchLabels:
          node-role.kubernetes.io/infra: ""
#### 確認ingress-operator已分配到Infra Node          
[devop@bastion ocp]$ oc get pod -n openshift-ingress -o wide
NAME                              READY   STATUS        RESTARTS   AGE     IP              NODE                  NOMINATED NODE   READINESS GATES
router-default-5c67b4db8b-75pjr   1/1     Running       0          5h44m   192.168.33.26   worker1.ocp.ken.lab   <none>           <none>
router-default-5c67b4db8b-glwcq   0/1     Terminating   0          5h44m   192.168.33.27   worker2.ocp.ken.lab   <none>           <none>
router-default-7c4cc569cc-85sbm   0/1     Pending       0          28s     <none>          <none>                <none>           <none>

[devop@bastion ocp]$ oc get pod -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE                 NOMINATED NODE   READINESS GATES
router-default-7c4cc569cc-86ggk   1/1     Running   0          18m   192.168.33.32   infra2.ocp.ken.lab   <none>           <none>
router-default-7c4cc569cc-97mlv   1/1     Running   0          49m   192.168.33.31   infra1.ocp.ken.lab   <none>           <none>

```

##### Registry

```
[devop@bastion ocp]$ oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
[devop@bastion ocp]$ oc edit configs.imageregistry.operator.openshift.io/cluster
spec:
  logLevel: Normal
  managementState: Removed
  observedConfig: null
  operatorLogLevel: Normal
  proxy: {}
  replicas: 1
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  rolloutStrategy: RollingUpdate
  storage: {}
  unsupportedConfigOverrides: null
  nodeSelector:
    node-role.kubernetes.io/infra: ""
[devop@bastion ocp]$ oc get pods -o wide -n openshift-image-registry
NAME                                               READY   STATUS      RESTARTS   AGE    IP              NODE                  NOMINATED NODE   READINESS GATES
cluster-image-registry-operator-598c5664bd-qz8cs   1/1     Running     0          23h    10.128.0.34     master1.ocp.ken.lab   <none>           <none>
image-pruner-27761760-jh4fz                        0/1     Completed   0          101m   10.131.2.8      infra3.ocp.ken.lab    <none>           <none>
node-ca-cdt8j                                      1/1     Running     0          22h    192.168.33.23   master1.ocp.ken.lab   <none>           <none>
node-ca-dpzzb                                      1/1     Running     1          22h    192.168.33.32   infra2.ocp.ken.lab    <none>           <none>
node-ca-hfv8c                                      1/1     Running     1          21h    192.168.33.33   infra3.ocp.ken.lab    <none>           <none>
node-ca-hjptk                                      1/1     Running     1          22h    192.168.33.31   infra1.ocp.ken.lab    <none>           <none>
node-ca-k2qhs                                      1/1     Running     0          22h    192.168.33.24   master2.ocp.ken.lab   <none>           <none>
node-ca-lgkg8                                      1/1     Running     0          22h    192.168.33.27   worker2.ocp.ken.lab   <none>           <none>
node-ca-wqpn8                                      1/1     Running     0          22h    192.168.33.25   master3.ocp.ken.lab   <none>           <none>
node-ca-xt8dg                                      1/1     Running     0          22h    192.168.33.26   worker1.ocp.ken.lab   <none>           <none>

```

##### Monitoring

Include: Prometheus, Thanos Querier, and Alertmanager
先建立ConfigMap cluster-monitor-config.yaml

```
[devop@bastion ocp]$ oc get pod -n openshift-monitoring -o wide
[devop@bastion ocp]$ vim cluster-monitoring-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    thanosQuerier:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
 --- 建立此ConfigMap
 [devop@bastion ocp]$ oc create -f cluster-monitoring-config.yaml 
configmap/cluster-monitoring-config created
--- 查看pod 是否都移轉至infra node，若未移轉則delete該pod，使其重新部署
node-exporter、cluster-monitoring-operator不用刪除
[devop@bastion ocp]$ watch oc get pod -n openshift-monitoring -o wide
[devop@bastion ocp]$ oc get pod -n openshift-monitoring -o wide |grep worker
[devop@bastion ocp]$ oc get pod -n openshift-monitoring -o wide |grep master
cluster-monitoring-operator-574c5f5b7b-sg6nh            2/2     Running   0          23h     10.128.0.33     master1.ocp.ken.lab   <none>           <none>
node-exporter-hr22c                                     2/2     Running   0          22h     192.168.33.23   master1.ocp.ken.lab   <none>           <none>
node-exporter-jnqkc                                     2/2     Running   0          22h     192.168.33.25   master3.ocp.ken.lab   <none>           <none>
node-exporter-lzk6n                                     2/2     Running   0          22h     192.168.33.24   master2.ocp.ken.lab   <none>           <none>
```



## 問題排除

### 調整

##### OperatorHub Blank

###### 畫面空白

![image-20221012115352528](https://kenkenny.synology.me:5543/images/2023/09/image-20221012115352528.png)

###### 在install-config.yaml 

裡面就要加入 marketplace的參數，sample是用來起template、image stream的operator
![image-20221012133723224](https://kenkenny.synology.me:5543/images/2023/09/image-20221012133723224.png)

###### install-config.yaml

```
[devop@bastion ocp]$ vim install-config.yaml 
[devop@bastion ocp]$ cat install-config.yaml 
apiVersion: v1
baseDomain: ken.lab 
compute: 
- hyperthreading: Enabled 
  name: worker
  replicas: 2 
controlPlane: 
  hyperthreading: Enabled 
  name: master
  replicas: 3 
metadata:
  name: ocp
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"}}}' 
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIN1Z2jXmGnPWGAVrtltN9A2q5D8QtJlRvhvdB2QXoZG devop@bastion.ocp.ken.lab' 
capabilities:
  baselineCapabilitySet: None
  additionalEnabledCapabilities:
  - marketplace
```



