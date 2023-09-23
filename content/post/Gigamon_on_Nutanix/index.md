---
title: Gigamon_on_Nutanix
description: Gigamon_on_Nutanix
slug: Gigamon_on_Nutanix
date: 2023-09-23T03:47:05+08:00
categories:
    - Lab Category
tags:
    - Nutanix
    - Gigamon
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Gigamon On Nutanix 



## 參考連結
https://docs.gigamon.com/pdfs/Content/Resources/PDF%20Library/GV-6100-Doc/GigaVUE-Cloud-Suite-Nutanix-GigaVUE-V-Series-2-Guide-v61.pdf
https://community.gigamon.com/gigamoncp/s/article/Deploying-Gigamon-Cloud-Suite-for-pervasive-visibility-on-Nutanix-underlay-using-native-service-chaining-6-1

AHV Networking

https://www.nutanixbible.com/5a-book-of-ahv-architecture.html
https://hyperhci.com/2019/11/20/nutanix-ahv-networking-advanced-features/

Fortinet
https://docs.fortinet.com/document/fortigate/6.4.0/new-features/932284/nutanix-service-chaining-6-4-5



## 架構圖
利用Nutanix的Flow Service Chaining 將流量導入到Gigamon vSeries
由GigaVUE-FM來控管要出去的流量,再從vSeries把流量導到資安設備 or Wireshark

![1.png](https://kenkenny.synology.me:5543/images/2023/09/rtaImage-20230519101724044.png)

## Port Requrements
![2.png](https://kenkenny.synology.me:5543/images/2023/09/rtaImage.png)

# 資訊

1. Gigamon FM
   https://172.16.101.160  admin / netfos123A!!



## Monitoring Mode

![image-20230519154853485](https://kenkenny.synology.me:5543/images/2023/09/image-20230519154853485.png)





## 依照手冊建立FM

## 建立vSeries

```
需注意Management跟Data Lan都要使用DHCP
也就是IPAM的子網
```

![image-20230519154627097](https://kenkenny.synology.me:5543/images/2023/09/image-20230519154627097.png)


![image-20230519163320235](https://kenkenny.synology.me:5543/images/2023/09/image-20230519163320235.png)

## tcpdump

1. 建立ubuntu VM

2. 安裝

   ```
   https://community.gigamon.com/gigamoncp/s/article/Deploying-Gigamon-Cloud-Suite-for-AWS-Behind-GWLB-to-Gain-Cross-Account-Visibility-6-1
   
   Once this session is successful, login to an Ubuntu instance being used as tools VM. User can input the following commands:
   
       Install TCPDump: sudo snap install tcpdump
       Retrieve traffic being sent by the VSeries node: tcpdump -i eth0 port 4789
       To capture traffic in a PCAP file instead of being displayed on terminal: sudo tcpdump -i eth0 port 4789 -w test.pcap
       (Optional) Download captured pcap to a local Linux workstation: scp test.pcap user-name@11.22.33.44:/Downloads
   
   The traffic being retrieve is VXLAN encapsulated. To de-capsulate the traffic:
   
       Add a VXLAN interface: sudo ip link add vxlan0 type vxlan id 112 group 239.1.1.1 dev eth0 dstport 4789
           The ID needs to match the VXLAN ID provided in the tunnel
       Bring up the VXLAN port: sudo ip link set vxlan0 up
       Capture the decapsulated packets: sudo tcpdump -ni vxlan0
   
   By following above steps 9 and 10, user can view the intended traffic from the source VMs as defined by the policies under the map.
   ```

   

## Wireshark

設定要監控的流量
![image-20230523161447891](https://kenkenny.synology.me:5543/images/2023/09/image-20230523161447891.png)

在FM調整tunnel 的 VxLan Network Identifier及Destination L4 Port
為 4789
![image-20230523161059697](https://kenkenny.synology.me:5543/images/2023/09/image-20230523161059697.png)

![image-20230523155647640](https://kenkenny.synology.me:5543/images/2023/09/image-20230523155647640.png)

![](https://kenkenny.synology.me:5543/images/2023/09/image-20230523155914647.png)

![image-20230523161239239](https://kenkenny.synology.me:5543/images/2023/09/image-20230523161239239.png)

![image-20230523160553714](https://kenkenny.synology.me:5543/images/2023/09/image-20230523160553714.png)

