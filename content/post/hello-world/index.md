---
title: Hello World
description: Welcome to Ken's blog
slug: hello-world
date: 2023-09-19
categories:
    - Lab Category
tags:
    - Lab
    - KB
    - Nutanix
    - Redhat
    - Vmware
    - Kubernetes
    - Openshift
    - Microsoft
    - SUSE
    - Nvidia
    - Python
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---



# Hello World



## Description

This is a blog for my labs and knowledge bases.

Happy engineering.

這是Ken的部落格，用來放我的Lab跟知識庫。

當個快樂的工程師。

![The Dangers of Hacking and What a Hacker Can Do to Your Computer? | by  Sravan Cynixit | Quick Code | Medium](https://kenkenny.synology.me:5543/images/2023/09/0*ngAthWxOvKZHvsw9.jpeg)

## This is a test shell script

Generate certificate script

```shell
#!/bin/bash
# Create certificate given a specific url name
if [ $# -ne 1 ]
  then
    echo "No arguments supplied or too many arguments" 
    echo "Usage: $0 domain_name"
    exit 1
fi

MYNAME=$1

echo "Generating a private key...for $MYNAME"
openssl genrsa -out $MYNAME.key 2048

echo "Generating a CSR...for $MYNAME"
openssl req -new -key $MYNAME.key \
-out $MYNAME.csr \
-subj "/C=US/ST=NC/L=Raleigh/O=RedHat/OU=RHT/CN=$MYNAME"

echo "Generating a certificate...for $MYNAME"
openssl x509 -req -days 366 -in \
$MYNAME.csr -signkey \
$MYNAME.key \
-out $MYNAME.crt
```

