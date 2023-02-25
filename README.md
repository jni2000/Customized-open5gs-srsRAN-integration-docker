# Open5GS in Docker
![Ubuntu package](https://img.shields.io/ubuntu/v/:v18.04.1-0ubuntu1)
![YouTube Video Likes](https://img.shields.io/youtube/likes/6pnh7TqTU6c?style=social)

This repository contains the following sections:
- Run Open5gs with user plane, control plane, combine eNB, UE with srsRAN
- Overview of Location Based Services system, using GMLC, E-SMLC
- Notes about Java, Spring Framework
- Note about installing and running JDK, maven, JDiameter

[Video instructions](https://www.youtube.com/watch?v=6pnh7TqTU6c)

## Contributors:
- [Ma Viet Duc](https://github.com/maduc238)
- [Pham Thanh Hai](https://github.com/ahihi8z8z)

### Table of contents

[1. Create communication gateway](#fire)

[2. Pull images from Docker Hub](#slong)

[3. Docker Run the received images](#slam)

[4. Run Containers and configure them](#sli)
- [4.1. User Plane](#slinung)
- [4.2. Part Control Plane](#slislong)
- [4.3. Part eNB, UE](#slislam)

[5. Work done](#ha)

<img src="images/Open5gs-config.jpg">

<a name="nung"></a>
## 1. Create communication gateway
Add gateway with ip 20.0.0.1
```
docker network create --gateway 20.0.0.1 --subnet 20.0.0.0/24 4g
```
Configuration diagram:
```
+-----------+        +---------------------+        +------------------+   if: ogs-internet 60.17.0.23/16
|  eNodeB   |        |  EPC Control Plane  |        |  EPC User Plane  |------------------------------------- INTERNET
+-----------+        +---------------------+        +------------------+
      |                         |                            |
      |  20.0.0.20              |  20.0.0.2,3,4              |  20.0.0.5,6                  if: 20.0.0.0/24
    --+-------------------------+----------------------------+--------------------------------------------------
```
Configurations of Docker containers:
| Docker # | Elements | IP Address | OS |
| --- | --- | --- | --- |
| EPC Control Plane | MME <br> SGW-C <br> SMF | 20.0.0.2/24 <br> 20.0.0.3/24 <br> 20.0.0.4/24 | Ubuntu 20.04 |
| EPC User Plane | SGW-U <br> UPF | 20.0.0.5/24 <br> 20.0.0.6/24 | Ubuntu 20.04 |
| srsRAN | eNodeB, UE | 20.0.0.20/24 | Ubuntu 20.04 |

Check the network for Docker with the command
```
docker network ls
```

<a name="slong"></a>
## 2. Pull images from Docker Hub
User Plane:
```
docker pull jni2000/open5gs-srsRAN-integration-docker:user-plane
```
Control Plane:
```
docker pull jni2000/open5gs-srsRAN-integration-docker:control-plane 
```
srsRAN
```
docker pull aothatday/open5gs:srsenb
```

<a name="slam"></a>
## 3. Docker Run pulled images
**Note: Two Docker images running on 2 different terminals**

User Plane: requires connection to the network, so it is necessary to create a virtual interface with mode tun
```
docker run --name open5gs-u -d -t --cap-add=NET_ADMIN --cap-add=NET_RAW --net 4g --ip 20.0.0.5 --device /dev/net/tun maduc238/open5gs:user-plane
```

Can run on main machine's network: `--network host`

Control Plane:
```
docker run --name open5gs-c -d -t --cap-add=NET_ADMIN --cap-add=NET_RAW --net 4g --ip 20.0.0.2 maduc238/open5gs:control-plane
```
Add the main machine connection port, for example: `-p 36412:36412/sctp`

srsRAN:
```
docker run --name srsenb -d -t --privileged -v /dev/bus/usb:/dev/bus/usb --net 4g --ip 20.0.0.20 aothatday/open5gs:srsenb
```

<a name="sli"></a>
## 4. Run Containers and configure them
<a name="slinung"></a>
### 4.1. User Plane
Configure the network and run the container:

Note: Need to adjust the ip of the interface S1-U (gtpu) and SGW-U: `vim install/etc/open5gs/sgwu.yaml`

```
docker exec -it open5gs-u bash 
```
```
ip addr add 20.0.0.6/24 dev eth0 
```
```
ip tuntap add name ogs-internet mode tun 
```
```
ip addr add 60.17.0.23/16 dev ogs-internet 
```
```
ip link set ogs-internet up 
```
```
iptables -t nat -A POSTROUTING -s 60.17.0.23 ! -o ogs-internet -j MASQUERADE 
```

**Lưu ý: Sửa IP nếu có chỉnh sửa trước khi chạy trong sgw-u**
```
cd home/open5gs 
./run.sh 
```
<a name="slislong"></a>
### 4.2. Control Plane
Configure the network and run the container:
```
docker exec -it open5gs-c bash 
```
```
ip addr add 20.0.0.3/24 dev eth0 
```
```
ip addr add 20.0.0.4/24 dev eth0 
```
```
cd open5gs 
./run4g_cp.sh 
```
Go to the UI web part:
Access address [20.0.0.2:3000](http://20.0.0.2:3000)
user name: `admin`
password: `1423`

<a name="slislam"></a>
### 4.3. eNB, UE
**Note: Run eNB and UE on 2 different terminals**

**eNB:**
```
docker exec -it srsenb bash
```
```
cd srsRAN/srsenb
../build/srsenb/src/srsenb ./enb.conf 
```
**With real eNB, edit `file /root/.config/srsran/enb.conf` and run `srsenb`**

**UE:**
```
docker exec -it srsenb bash
```
```
cd srsRAN/srsue
../build/srsue/src/srsue ./ue.conf
```

<a name="ha"></a>
## 5. Work done - verification
Get the id of the 4g subnet created at the beginning
```
docker network ls | grep 4g
```
Docker host: capture packets passing through docker's newly created interface of the form br-id
```
sudo wireshark
```
Container machines: use tcpdump to capture packets in the loopback interface
```
tcpdump -i lo -s 65535 -w loopback.pcap
```
