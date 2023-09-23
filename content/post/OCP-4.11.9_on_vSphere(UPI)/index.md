---
title: OCP-4.11.9_on_vSphere(UPI)
description: OCP-4.11.9_on_vSphere(UPI)
slug: OCP-4.11.9_on_vSphere(UPI)
date: 2023-09-23T08:19:11+08:00
categories:
    - Lab Category
tags:
    - Openshift
    - Redhat
    - Kubernetes
    - Vmware
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Install Openshift 4.11.9 on vSphere(UPI)

### 說明

https://docs.openshift.com/container-platform/4.11/installing/installing_vsphere/installing-vsphere.html
https://cloud.redhat.com/blog/deploying-openshift-4.4-to-vmware-vsphere-7?hsLang=en-us

1. 用Mac 當作Client
2. 需要事先安裝好Bastion
3. Bastion安裝設定Named、HAProxy、Web Server (這裡用Nginx)
4. 用rhcos-4.11.9.ova 來ESXi 部署ovf 範本

### 下載OC Client

https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.11.9/openshift-client-mac-4.11.9.tar.gz

### 下載OCP Installer

https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.11.9/openshift-install-mac-4.11.9.tar.gz

### 下載CoreOS-4.11.9.ova

https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.11/4.11.9/rhcos-4.11.9-x86_64-vmware.x86_64.ova

### 下載Pull-Secret

https://console.redhat.com/openshift/downloads#tool-pull-secret

### Named

```
Named 設定檔
[root@bastion ~]# cat /etc/named.conf

//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
	listen-on port 53 { any; };
#	listen-on-v6 port 53 { ::1; };
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

	dnssec-enable yes;
	dnssec-validation yes;

	managed-keys-directory "/var/named/dynamic";

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



zone "ocp.infotech.com.tw" IN {
type master;
file "/var/named/forward.bastion.ocp";
allow-update { none; };
allow-query { any; };
allow-transfer { none; };
};
zone "3.168.192.in-addr.arpa" IN {
type master;
file "/var/named/reverse.bastion.ocp";
allow-update { none; };
allow-query { any; };
allow-transfer { none; };
};


include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

------ 正向解析
[root@bastion ~]# cat /var/named/forward.bastion.ocp 
$TTL 1W
@	IN	SOA	ns1.infotech.com.tw.	root (
			2019070700	; serial
			3H		; refresh (3 hours)
			30M		; retry (30 minutes)
			2W		; expiry (2 weeks)
			1W )		; minimum (1 week)
	IN	NS	ns1.infotech.com.tw.
	IN	MX 10	smtp.infotech.com.tw.
;
;
ns1.infotech.com.tw.		IN	A	192.168.3.37
smtp.infotech.com.tw.		IN	A	192.168.3.37
;
helper.infotech.com.tw.		IN	A	192.168.3.37
helper.ocp.infotech.com.tw.	IN	A	192.168.3.37
;
api.ocp.infotech.com.tw.	IN	A	192.168.3.37 
api-int.ocp.infotech.com.tw.	IN	A	192.168.3.37 
;
*.apps.ocp.infotech.com.tw.	IN	A	192.168.3.37 
;
bootstrap.ocp.infotech.com.tw.	IN	A	192.168.3.36 
;
master1.ocp.infotech.com.tw.	IN	A	192.168.3.31 
master2.ocp.infotech.com.tw.	IN	A	192.168.3.32 
master3.ocp.infotech.com.tw.	IN	A	192.168.3.33 
;
worker1.ocp.infotech.com.tw.	IN	A	192.168.3.34 
worker2.ocp.infotech.com.tw.	IN	A	192.168.3.35
;
;EOF

------- 反向解析
[root@bastion ~]# cat /var/named/reverse.bastion.ocp 
$TTL 1W
@	IN	SOA	ns1.infotech.com.tw.	root (
			2019070700	; serial
			3H		; refresh (3 hours)
			30M		; retry (30 minutes)
			2W		; expiry (2 weeks)
			1W )		; minimum (1 week)
	IN	NS	ns1.infotech.com.tw.
;
37.3.168.192.in-addr.arpa.	IN	PTR	api.ocp.infotech.com.tw. 
37.3.168.192.in-addr.arpa.	IN	PTR	api-int.ocp.infotech.com.tw. 
;
36.3.168.192.in-addr.arpa.	IN	PTR	bootstrap.ocp.infotech.com.tw. 
;
31.3.168.192.in-addr.arpa.	IN	PTR	master1.ocp.infotech.com.tw. 
32.3.168.192.in-addr.arpa.	IN	PTR	master2.ocp.infotech.com.tw. 
33.3.168.192.in-addr.arpa.	IN	PTR	master3.ocp.infotech.com.tw. 
;
34.3.168.192.in-addr.arpa.	IN	PTR	worker1.ocp.infotech.com.tw. 
35.3.168.192.in-addr.arpa.	IN	PTR	worker2.ocp.infotech.com.tw. 
;
;EOF
```

### HAProxy

```
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
  stats show-desc Stats for ocp cluster 
  stats auth admin:ocp
  stats uri /stats
listen api-server-6443 
  bind *:6443
  mode tcp
  server bootstrap bootstrap.ocp.infotech.com.tw:6443 check inter 1s backup 
  server master1 master1.ocp.infotech.com.tw:6443 check inter 1s
  server master2 master2.ocp.infotech.com.tw:6443 check inter 1s
  server master3 master3.ocp.infotech.com.tw:6443 check inter 1s
listen machine-config-server-22623 
  bind *:22623
  mode tcp
  server bootstrap bootstrap.ocp.infotech.com.tw:22623 check inter 1s backup 
  server master1 master1.ocp.infotech.com.tw:22623 check inter 1s
  server master2 master2.ocp.infotech.com.tw:22623 check inter 1s
  server master3 master3.ocp.infotech.com.tw:22623 check inter 1s
listen ingress-router-443 
  bind *:443
  mode tcp
  balance source
  server worker1 worker1.ocp.infotech.com.tw:443 check inter 1s
  server worker2 worker2.ocp.infotech.com.tw:443 check inter 1s
listen ingress-router-80 
  bind *:80
  mode tcp
  balance source
  server worker1 worker1.ocp.infotech.com.tw:80 check inter 1s
  server worker2 worker2.ocp.infotech.com.tw:80 check inter 1s
```

### Nginx

```
根目錄 /usr/share/nginx/html
Port: 8000

[root@bastion html]# cat /etc/nginx/nginx.conf
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       8000 default_server;
        listen       [::]:8000 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers PROFILE=SYSTEM;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }

}


```



## 產生安裝檔案

### 確認 cli tool

```
$ ls -l
total 2232728
drwxr-xr-x   8 wangken  staff        256 12 13 15:36 .
drwxr-x---+ 41 wangken  staff       1312 12 13 15:35 ..
-rw-r--r--@  1 wangken  staff        706 10  8 18:41 README.md
-rwxr-xr-x@  2 wangken  staff   98436592 10  6 20:06 kubectl
-rwxr-xr-x@  2 wangken  staff   98436592 10  6 20:06 oc
-rw-r--r--@  1 wangken  staff   44116650 12 13 15:06 openshift-client-mac-4.11.9.tar.gz
-rwxr-xr-x@  1 wangken  staff  528956912 10  8 18:41 openshift-install
-rw-r--r--@  1 wangken  staff  363543317 12 13 15:07 openshift-install-mac-4.11.9.tar.gz

$ ./oc version
Client Version: 4.11.9
Kustomize Version: v4.5.4

$ ./openshift-install version
./openshift-install 4.11.9
built from commit 01a6869a6f1208fb4d112060c5971432fdd619cf
release image quay.io/openshift-release-dev/ocp-release@sha256:94b611f00f51c9acc44ca3f4634e46bd79d7d28b46047c7e3389d250698f0c99
release architecture amd64
```

### 產生SSH key (Bastion)

```
[root@bastion .ssh]# cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDF/jswkiYiL+jXDMMUpwga7CGV79kXtcMnOpbzO5xOwQAAAKjY1yrT2Ncq
0wAAAAtzc2gtZWQyNTUxOQAAACDF/jswkiYiL+jXDMMUpwga7CGV79kXtcMnOpbzO5xOwQ
AAAEBIWOmAtlitDdZXhb+1vtMnSbNBRQfyANPR/eHUPuisK8X+OzCSJiIv6NcMwxSnCBrs
IZXv2Re1wyc6lvM7nE7BAAAAIWRldm9wQGJhc3Rpb24ub2NwLmluZm90ZWNoLmNvbS50dw
ECAwQ=
-----END OPENSSH PRIVATE KEY-----
[root@bastion .ssh]# cat id_rsa.pub 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMX+OzCSJiIv6NcMwxSnCBrsIZXv2Re1wyc6lvM7nE7B devop@bastion.ocp.infotech.com.tw

```

### 建立Install-config.yaml

```

apiVersion: v1
baseDomain: infotech.com.tw
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
  vsphere:
    vcenter: tpeinfovcsa7p.infotech.com.tw
    username: cisadmin@vsphere.local
    password: P@55w.rd
    datacenter: TPECIS_Datacenter
    defaultDatastore: v7000_CIS01
    diskType: thin  
fips: false 
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfMzEwNjVjNmVmMTk5NDZmNWEwNmUzZmI4MTVmNzk2OTE6NEJRVk1FVk9CTDRZQVRBT0pONEhQVkIyUExFQTdKNDE3WExFU1IzVDk3NU1YUTJXWE9FSjBNTVk0MzFQQk9RUA==","email":"ken.wang@infotech.com.tw"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTc5MzgwY2YyLTUxNTEtNGRiNi04YjA3LTRiNzQ4NDcxNWE2NDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmxOak5rWVdWbU1tSXlaRFEwTVdKaVlUTXpObVUwTkdGbVl6WmtPVEUwWVNKOS5RMlp6Q1JaWWltVDIxM2dpYU9CRWtNWEVkYUtjbnc4WHN1bTY3VEVOdjEyRjBjQUgyNWNZMVdWTjZNUEY2R21CNjh3TmF4TnoxRHVpVnBtY0ZKM2JZTW0yNVVzZGJxeFc4cEd1dGszSzF5NlNUWk9VejlkU0V4UGg4aUxha05qSEQ4Tml6emNmM1lUQjRNUXdhYTVFSG1PMXVTUERUZFRBTXFYS3FFa2gwRXpHOUpKZTlUMnM4bVJXWkVIbndoZW84cktncVZuV1VCeGhqTldFNElfWG1JZnBPRl91akpFNHdXNzRJWjd5UmdoUGExQVVwS3BsY0NJQ2ZGdGtaS29WUTFwNTBzUzNabkxWYjd6ZHdXR21WcXVsNUFoZUNiSHNoNEtpSjFnYXRSbkwtMG92THZoNGdseUZqd0c0Z3lMMEdyS1ItbU5WMFZYc01mVzJDNXNVaXBzVmhKcmhmY01vQjRNSDRDbjZKcjJXS2plbEdqSVdHQzBUV1BBNlZQZDBmc0lyLTRCY2RBdUFmcm5SaWpGM3d0Mng3RFdXQ1RScFVPbUVMY3U5bDR6Wm5uZkVSejNMd01rNmhCWTFyYXV1TGhCWXJraFlzQURqWEl3bGxPTkwyc01GMUU2OVNIVG1vV0tvNU5fdHdoWE5fX1BtT1ozMzc5ZVNDVGdnTFNZd3hyR1VwS3FXRDFva3JWZlRIWmNlcTk4WHQ2QjFtZHIxcmxLV2RiRmhsRDQ3Q1pldEhrNW1xQUlianp1UzlONENJWllhd0lIZE5LWVotN2tRNFQyUFg4QWdadHFNdHdjY2c0bWx3ME40c09LbWVZN2Q2bGw0bXpXV201bVo0aktTeHdpYmJ3STEwOFcxWVFlSlQ4QW9WZFlkc09YWkdWc3ZxZVVZOU9qbzR2RQ==","email":"ken.wang@infotech.com.tw"}}}'
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMX+OzCSJiIv6NcMwxSnCBrsIZXv2Re1wyc6lvM7nE7B devop@bastion.ocp.infotech.com.tw' 
capabilities:
  baselineCapabilitySet: vCurrent
  
-------------  
    platform:
    vsphere:
      vcenter: tpeinfovcsa7p.infotech.com.tw
      username: cisadmin@vsphere.local
      password: P@55w.rd
      datacenter: TPECIS_Datacenter
      defaultDatastore: v7000_CIS01
      diskType: thin  
```

### 產生Ignition file

```
./openshift-install create manifests --dir install 
./openshift-install create ignition-configs --dir install

INFO Consuming Openshift Manifests from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming Worker Machines from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Ignition-Configs created in: install and install/auth 

cd install 
$ ll
total 2960
drwxr-xr-x   9 wangken  staff      288 12 13 16:27 .
drwxr-xr-x  11 wangken  staff      352 12 13 15:56 ..
-rw-r--r--   1 wangken  staff    67239 12 13 16:27 .openshift_install.log
-rw-r-----   1 wangken  staff  1154395 12 13 16:27 .openshift_install_state.json
drwxr-x---   4 wangken  staff      128 12 13 16:27 auth
-rw-r-----   1 wangken  staff   275583 12 13 16:27 bootstrap.ign
-rw-r-----   1 wangken  staff     1721 12 13 16:27 master.ign
-rw-r-----   1 wangken  staff       94 12 13 16:27 metadata.json
-rw-r-----   1 wangken  staff     1721 12 13 16:27 worker.ign

```

### 複製Ignition File 到Web

```
scp *.ign root@192.168.3.37:/usr/share/nginx/html/
root@192.168.3.37's password: 
bootstrap.ign                                 100%  269KB   3.5MB/s   00:00    
master.ign                                    100% 1721    86.9KB/s   00:00    
worker.ign                                    100% 1721   188.5KB/s   00:00 

-----------
cat merge-bootstrap.ign 
{
  "ignition": {
    "config": {
      "merge": [
        {
          "source": "http://192.168.3.37:8000/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "3.2.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}


-----轉檔為Base64
chmod 644 *.ign
[root@bastion html]# base64 -w0 merge-bootstrap.ign > merge-bootstrap.64
ewogICJpZ25pdGlvbiI6IHsKICAgICJjb25maWciOiB7CiAgICAgICJtZXJnZSI6IFsKICAgICAgICB7CiAgICAgICAgICAic291cmNlIjogImh0dHA6Ly8xOTIuMTY4LjMuMzc6ODAwMC9ib290c3RyYXAuaWduIiwKICAgICAgICAgICJ2ZXJpZmljYXRpb24iOiB7fQogICAgICAgIH0KICAgICAgXQogICAgfSwKICAgICJ0aW1lb3V0cyI6IHt9LAogICAgInZlcnNpb24iOiAiMy4yLjAiCiAgfSwKICAibmV0d29ya2QiOiB7fSwKICAicGFzc3dkIjoge30sCiAgInN0b3JhZ2UiOiB7fSwKICAic3lzdGVtZCI6IHt9Cn0K

[root@bastion html]# base64 -w0 master.ign > master.64
[root@bastion html]# base64 -w0 worker.ign > worker.64

```



## 複製OVF至VM

### 調整參數

```
vApp
1. base64
2. 
bootstrap
ewogICJpZ25pdGlvbiI6IHsKICAgICJjb25maWciOiB7CiAgICAgICJtZXJnZSI6IFsKICAgICAgICB7CiAgICAgICAgICAic291cmNlIjogImh0dHA6Ly8xOTIuMTY4LjMuMzc6ODAwMC9ib290c3RyYXAuaWduIiwKICAgICAgICAgICJ2ZXJpZmljYXRpb24iOiB7fQogICAgICAgIH0KICAgICAgXQogICAgfSwKICAgICJ0aW1lb3V0cyI6IHt9LAogICAgInZlcnNpb24iOiAiMy4yLjAiCiAgfSwKICAibmV0d29ya2QiOiB7fSwKICAicGFzc3dkIjoge30sCiAgInN0b3JhZ2UiOiB7fSwKICAic3lzdGVtZCI6IHt9Cn0K

master
eyJpZ25pdGlvbiI6eyJjb25maWciOnsibWVyZ2UiOlt7InNvdXJjZSI6Imh0dHBzOi8vYXBpLWludC5vY3AuaW5mb3RlY2guY29tLnR3OjIyNjIzL2NvbmZpZy9tYXN0ZXIifV19LCJzZWN1cml0eSI6eyJ0bHMiOnsiY2VydGlmaWNhdGVBdXRob3JpdGllcyI6W3sic291cmNlIjoiZGF0YTp0ZXh0L3BsYWluO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVVJGUkVORFFXWnBaMEYzU1VKQlowbEpXV1JoZUZwMFZXVkVha2wzUkZGWlNrdHZXa2xvZG1OT1FWRkZURUpSUVhkS2FrVlRUVUpCUjBFeFZVVUtRM2hOU21JelFteGliazV2WVZkYU1FMVNRWGRFWjFsRVZsRlJSRVYzWkhsaU1qa3dURmRPYUUxQ05GaEVWRWw1VFZSSmVFMTZRVFJPUkZVeFRteHZXQXBFVkUxNVRWUkplRTFFUVRST1JGVXhUbXh2ZDBwcVJWTk5Ra0ZIUVRGVlJVTjRUVXBpTTBKc1ltNU9iMkZYV2pCTlVrRjNSR2RaUkZaUlVVUkZkMlI1Q21JeU9UQk1WMDVvVFVsSlFrbHFRVTVDWjJ0eGFHdHBSemwzTUVKQlVVVkdRVUZQUTBGUk9FRk5TVWxDUTJkTFEwRlJSVUV6U0hWcVNHc3haaTlCWWxNS2FHcEdhemxtV0hOelZqZFJNVzlJUVRkVFNqQnVlVkpqTDJNMmR6aERPV1ZUV1dkTWNIWnJjSEZEY3pCdFR5dE9NMHRGTW5Kb2VTdDVVVUZyUVN0eEt3cE5TVkJXVURWRU9HRjFkM3BwWTFOUFZERm5WR2xTTmpreWFFVTFVVEUwYmtaS1N6bE5UMGRaWkdJeFJFTklNazlJYkZkMWRqRktlVlJvUW5admJuRXhDamw2VG01SldreG5iMWRUUm5SWGNtMVFhR05vY3pJeE5sVTVhREpDY0U1QmIwbzJXbU01TjBZM09UWnRLMFpLVlVKeGRFSnhVMVU1UW0xMmEwbHZSa3dLVFd0dE5HczVORk50UzFCRVpURnpPSEZGY0ZGM2FDdDVhbGRPV214aFkyeHRaMGRTTVdkRFptbGxPRGt3UjJ4dWRUazFSbWMwWjBORVFXMHpNRkZ4V2dvdk1tVkNXVEJ2ZW5kMFkyOTZVakZVVkU1cVpUSnpXaTlPZEhBMUwyRTBiR3RMYjNweGJXSnpURE13U25aaFNtbzVlVmh4T1VRelZtRnZVV2xIWkhWbENsQkpMMlJ6WkZsTWEzZEpSRUZSUVVKdk1FbDNVVVJCVDBKblRsWklVVGhDUVdZNFJVSkJUVU5CY1ZGM1JIZFpSRlpTTUZSQlVVZ3ZRa0ZWZDBGM1JVSUtMM3BCWkVKblRsWklVVFJGUm1kUlZXdEljR1IyVG5sUlVuZzFkVWhoTjJoWmJ6Tm9RM0Y2ZDJ4M1VYZEVVVmxLUzI5YVNXaDJZMDVCVVVWTVFsRkJSQXBuWjBWQ1FVbzVlRnBRTlRNeVZqWjBaa3RVVGxSVGFGTllWbEZVTlRCdmF6VkRUR3h1ZVVaNWRtRkNjV2huZEhwVlZsZGpPR2xGVTFocWFYbHJSRlZxQ2twallXcFZOSEZaU1ZkSmVGWlJNVmR3Um5GcFNqQlpkbTV4TXpOa2JFMUdORTg1U1VWT1owTXdkRGx3UjI1R1NGSlpLMUV6YjA5WVkzUXdWRlF2UkhVS1JuSnpXbFpoTTJsVE1rZDRaSGc1ZDFsV1pFbHVUVzV2V0VscFFrRlliMDlaTTBaNVYwRlROakJOV1RkMmVXTkhObXRLVFZWdVpUbDRVbGRyWjBsRFJBcDZOVXBXU0RSUE1FZ3pZVGxaTVhsSFowODBTVVpSVVN0c1JsZENiMmhOU2xrelpXaEpXR05oVDJORGJsWTNZV1pvUXpKcVVHbFpPRTFrZFdsWU9HRkRDblExUkdVM1psQjRUMmsxVWpSclZWQmtWRFYwYkhGeldHdEliWEJFU3lzM1ZVaFZLMk13ZFdGd1FVTjVObVUzU1U1U1dGbHViVkpMTlVadFJUSkZPVVFLUmxoc2NXdEtTM1JITVVoVU5rSXdZbnB5UzBsU05ucDVPVVJyUFFvdExTMHRMVVZPUkNCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2c9PSJ9XX19LCJ2ZXJzaW9uIjoiMy4yLjAifX0=

worker
eyJpZ25pdGlvbiI6eyJjb25maWciOnsibWVyZ2UiOlt7InNvdXJjZSI6Imh0dHBzOi8vYXBpLWludC5vY3AuaW5mb3RlY2guY29tLnR3OjIyNjIzL2NvbmZpZy93b3JrZXIifV19LCJzZWN1cml0eSI6eyJ0bHMiOnsiY2VydGlmaWNhdGVBdXRob3JpdGllcyI6W3sic291cmNlIjoiZGF0YTp0ZXh0L3BsYWluO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LExTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVVJGUkVORFFXWnBaMEYzU1VKQlowbEpXV1JoZUZwMFZXVkVha2wzUkZGWlNrdHZXa2xvZG1OT1FWRkZURUpSUVhkS2FrVlRUVUpCUjBFeFZVVUtRM2hOU21JelFteGliazV2WVZkYU1FMVNRWGRFWjFsRVZsRlJSRVYzWkhsaU1qa3dURmRPYUUxQ05GaEVWRWw1VFZSSmVFMTZRVFJPUkZVeFRteHZXQXBFVkUxNVRWUkplRTFFUVRST1JGVXhUbXh2ZDBwcVJWTk5Ra0ZIUVRGVlJVTjRUVXBpTTBKc1ltNU9iMkZYV2pCTlVrRjNSR2RaUkZaUlVVUkZkMlI1Q21JeU9UQk1WMDVvVFVsSlFrbHFRVTVDWjJ0eGFHdHBSemwzTUVKQlVVVkdRVUZQUTBGUk9FRk5TVWxDUTJkTFEwRlJSVUV6U0hWcVNHc3haaTlCWWxNS2FHcEdhemxtV0hOelZqZFJNVzlJUVRkVFNqQnVlVkpqTDJNMmR6aERPV1ZUV1dkTWNIWnJjSEZEY3pCdFR5dE9NMHRGTW5Kb2VTdDVVVUZyUVN0eEt3cE5TVkJXVURWRU9HRjFkM3BwWTFOUFZERm5WR2xTTmpreWFFVTFVVEUwYmtaS1N6bE5UMGRaWkdJeFJFTklNazlJYkZkMWRqRktlVlJvUW5admJuRXhDamw2VG01SldreG5iMWRUUm5SWGNtMVFhR05vY3pJeE5sVTVhREpDY0U1QmIwbzJXbU01TjBZM09UWnRLMFpLVlVKeGRFSnhVMVU1UW0xMmEwbHZSa3dLVFd0dE5HczVORk50UzFCRVpURnpPSEZGY0ZGM2FDdDVhbGRPV214aFkyeHRaMGRTTVdkRFptbGxPRGt3UjJ4dWRUazFSbWMwWjBORVFXMHpNRkZ4V2dvdk1tVkNXVEJ2ZW5kMFkyOTZVakZVVkU1cVpUSnpXaTlPZEhBMUwyRTBiR3RMYjNweGJXSnpURE13U25aaFNtbzVlVmh4T1VRelZtRnZVV2xIWkhWbENsQkpMMlJ6WkZsTWEzZEpSRUZSUVVKdk1FbDNVVVJCVDBKblRsWklVVGhDUVdZNFJVSkJUVU5CY1ZGM1JIZFpSRlpTTUZSQlVVZ3ZRa0ZWZDBGM1JVSUtMM3BCWkVKblRsWklVVFJGUm1kUlZXdEljR1IyVG5sUlVuZzFkVWhoTjJoWmJ6Tm9RM0Y2ZDJ4M1VYZEVVVmxLUzI5YVNXaDJZMDVCVVVWTVFsRkJSQXBuWjBWQ1FVbzVlRnBRTlRNeVZqWjBaa3RVVGxSVGFGTllWbEZVTlRCdmF6VkRUR3h1ZVVaNWRtRkNjV2huZEhwVlZsZGpPR2xGVTFocWFYbHJSRlZxQ2twallXcFZOSEZaU1ZkSmVGWlJNVmR3Um5GcFNqQlpkbTV4TXpOa2JFMUdORTg1U1VWT1owTXdkRGx3UjI1R1NGSlpLMUV6YjA5WVkzUXdWRlF2UkhVS1JuSnpXbFpoTTJsVE1rZDRaSGc1ZDFsV1pFbHVUVzV2V0VscFFrRlliMDlaTTBaNVYwRlROakJOV1RkMmVXTkhObXRLVFZWdVpUbDRVbGRyWjBsRFJBcDZOVXBXU0RSUE1FZ3pZVGxaTVhsSFowODBTVVpSVVN0c1JsZENiMmhOU2xrelpXaEpXR05oVDJORGJsWTNZV1pvUXpKcVVHbFpPRTFrZFdsWU9HRkRDblExUkdVM1psQjRUMmsxVWpSclZWQmtWRFYwYkhGeldHdEliWEJFU3lzM1ZVaFZLMk13ZFdGd1FVTjVObVUzU1U1U1dGbHViVkpMTlVadFJUSkZPVVFLUmxoc2NXdEtTM1JITVVoVU5rSXdZbnB5UzBsU05ucDVPVVJyUFFvdExTMHRMVVZPUkNCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2c9PSJ9XX19LCJ2ZXJzaW9uIjoiMy4yLjAifX0=
```



#### 設定IP

```
guestinfo.afterburn.initrd.network-kargs
ip=192.168.3.36::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37 
ip=192.168.3.31::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37
ip=192.168.3.32::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37
ip=192.168.3.33::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37
ip=192.168.3.34::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37
ip=192.168.3.35::192.168.3.251:255.255.255.0:::none nameserver=192.168.3.37
```



## 先啟動bootstrap

## 再啟動Master

## 啟動Worker後 Approve CSR

```
$oc adm certificate approve <csr_name>
----
$oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
```

## Storage Class 

https://access.redhat.com/solutions/4618011

https://access.redhat.com/solutions/5900501
https://www.ibm.com/support/pages/creating-persistent-volume-claim-fails-not-found-openshift
https://docs.openshift.com/container-platform/4.9/storage/persistent_storage/persistent-storage-vsphere.html

https://github.com/rancher/rancher/issues/24258

on vmware vsphere

![image-20221214141836955](https://kenkenny.synology.me:5543/images/2023/09/image-20221214141836955.png)

```
[devop@bastion ~]$ oc get machineset ocp-7hb6g-worker -n openshift-machine-api -o yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  annotations:
    machine.openshift.io/memoryMb: "16384"
    machine.openshift.io/vCPU: "4"
  creationTimestamp: "2022-12-13T09:05:14Z"
  generation: 1
  labels:
    machine.openshift.io/cluster-api-cluster: ocp-7hb6g
  name: ocp-7hb6g-worker
  namespace: openshift-machine-api
  resourceVersion: "12244"
  uid: e52a023d-bc9f-4fb5-a66a-776768e1aa7c
spec:
  replicas: 0
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ocp-7hb6g
      machine.openshift.io/cluster-api-machineset: ocp-7hb6g-worker
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ocp-7hb6g
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ocp-7hb6g-worker
    spec:
      lifecycleHooks: {}
      metadata: {}
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1beta1
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          kind: VSphereMachineProviderSpec
          memoryMiB: 16384
          metadata:
            creationTimestamp: null
          network:
            devices:
            - networkName: ""
          numCPUs: 4
          numCoresPerSocket: 4
          snapshot: ""
          template: ocp-7hb6g-rhcos
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: TPECIS_Datacenter
            datastore: v7000_CIS01
            folder: /TPECIS_Datacenter/vm/ocp-7hb6g
            resourcePool: /TPECIS_Datacenter/host//Resources
            server: tpeinfovcsa7p.infotech.com.tw
status:
  observedGeneration: 1
  replicas: 0


[devop@bastion ~]$ oc get cm cloud-provider-config  -n openshift-config -o yaml | grep folder
    folder = "/TPECIS_Datacenter/vm/ocp-7hb6g"
```

![image-20221214151017424](https://kenkenny.synology.me:5543/images/2023/09/image-20221214151017424.png)

![image-20221214151029390](https://kenkenny.synology.me:5543/images/2023/09/image-20221214151029390.png)

![image-20221214151038817](https://kenkenny.synology.me:5543/images/2023/09/image-20221214151038817.png)

![image-20221214151215733](https://kenkenny.synology.me:5543/images/2023/09/image-20221214151215733.png)
