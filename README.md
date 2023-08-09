# VoLTE and SMS Configuration for docker_open5gs
This briefly describes the settings for using **VoLTE** and **SMS** of [docker_open5gs](https://github.com/herlesupreeth/docker_open5gs).

---

<h2 id="conf_list">List of Sample Configurations</h2>

1. [One SGW-C/PGW-C, one SGW-U/PGW-U and one APN](https://github.com/s5uishida/open5gs_epc_srsran_sample_config)
2. [One SGW-C/PGW-C, Multiple SGW-Us/PGW-Us and APNs](https://github.com/s5uishida/open5gs_epc_oai_sample_config)
3. [One SMF, Multiple UPFs and DNNs](https://github.com/s5uishida/open5gs_5gc_ueransim_sample_config)
4. [Select nearby UPF(PGW-U) according to the connected eNodeB](https://github.com/s5uishida/open5gs_epc_srsran_nearby_upf_sample_config)
5. [Select nearby UPF according to the connected gNodeB](https://github.com/s5uishida/open5gs_5gc_ueransim_nearby_upf_sample_config)
6. [Select UPF based on S-NSSAI](https://github.com/s5uishida/open5gs_5gc_ueransim_snssai_upf_sample_config)
7. [SCP Indirect communication Model C](https://github.com/s5uishida/open5gs_5gc_ueransim_scp_model_c_sample_config)
8. VoLTE and SMS Configuration for docker_open5gs (this article)
9. [Monitoring Metrics with Prometheus](https://github.com/s5uishida/open5gs_5gc_ueransim_metrics_sample_config)
10. [Framed Routing](https://github.com/s5uishida/open5gs_5gc_ueransim_framed_routing_sample_config)
11. [VPP-UPF(PGW-U) with DPDK](https://github.com/s5uishida/open5gs_epc_srsran_vpp_upf_dpdk_sample_config)
12. [VPP-UPF with DPDK](https://github.com/s5uishida/open5gs_5gc_ueransim_vpp_upf_dpdk_sample_config)
---

<h2 id="misc">Miscellaneous Notes</h2>

- [Install MongoDB 6.0 and Open5GS WebUI](https://github.com/s5uishida/open5gs_install_mongodb6_webui)
- [Install MongoDB 4.4.18 on Ubuntu 20.04 for Raspberry Pi 4B](https://github.com/s5uishida/install_mongodb_on_ubuntu_for_rp4b)
- [A Note for Changing Network Interface of UPF from TUN to TAP in Open5GS](https://github.com/s5uishida/change_from_tun_to_tap_in_open5gs)
---
<h2 id="toc">Table of Contents</h2>

- [Overview of Network Configuration](#overview)
- [Use open5gs_hss_cx branch of docker_open5gs](#branch_open5gs)
  - [Changes in configuration files of docker_open5gs](#change_config)
  - [Build docker image for Open5GS and Kamailio](#build)
  - [Run docker-compose](#run)
  - [Register subscribers information with Open5GS](#register_open5gs)
  - [Register the IMSI and MSISDN with OsmoHLR](#register_osmocom)
  - [Try VoLTE and SMS](#try)
    - [Send SMS from OsmoMSC VTY terminal (SMS over SGs)](#osmomsc_send_command)
- [VoLTE and SMS with Raspberry Pi 4B](#rp4)
  - [Change how to create the docker image and container of MongoDB](#rp4_mongodb)
- [Trouble Information](#trouble)
- [Changelog (summary)](#changelog)
---

<h2 id="overview">Overview of Network Configuration</h2>

This describes the following setting example.
```
     eNodeB                Docker host - 172.22.0.0/24
    ---------             -------------
   |         |           |             |
   |         |           |             |
    ---------             -------------
        |                       |
-------------------------------------------------------
       .10                     .20      192.168.0.0/24
```
For reference, the Docker host VM on VirtualBox I have tried is as follows:
| CPU Cores | Memory | SSD | OS |
| --- | --- | --- | --- |
| 2 | 4GB | 40GB | Ubuntu 20.04 |

For eNodeB, set static routing to eNodeB for packets going from eNodeB to the Docker host network (`172.22.0.0/24`).
```
# ip route add 172.22.0.0/24 via 192.168.0.20
```
If eNodeB does not have a static routing setting function, build a network so that the Docker host becomes the default GW.

The PDNs are as follows.
| PDN | TUNnel interface of PDN | APN | U-Plane # |
| --- | --- | --- | --- |
| 192.168.100.0/24 | ogstun | internet | U-Plane1 |
| 192.168.101.0/24 | ogstun2 | ims | U-Plane1 |

<h2 id="branch_open5gs">Use open5gs_hss_cx branch of docker_open5gs</h2>

For **VoLTE** and **SMS over IMS** / **SMS over SGs**, it is convenient to use `open5gs_hss_cx` branch.
Kamailio's S-CSCF and I-CSCF communicate with Open5GS HSS(Cx).  
**Note.** For **SMS over SGs** of docker_open5gs, 2G/3G radio access is not used at all rather it uses only the core network (MSC + SMSC + HLR).
```
# git clone https://github.com/herlesupreeth/docker_open5gs
# cd docker_open5gs
# git checkout open5gs_hss_cx
```

<h3 id="change_config">Changes in configuration files of docker_open5gs</h3>

**.env)**

| Item | Value |
| --- | --- |
| MCC | 001 |
| MNC | 01 |
| DOCKER_HOST_IP | 192.168.0.20 |
| SGWU_ADVERTISE_IP | 192.168.0.20 |

**4g-volte-deploy.yaml)**

Publish the following ports of SGW-U and MME.
```diff
--- 4g-volte-deploy.yaml.orig   2023-08-07 10:58:37.018266553 +0000
+++ 4g-volte-deploy.yaml        2023-08-07 11:00:21.227760253 +0000
@@ -100,8 +100,8 @@
     expose:
       - "8805/udp"
       - "2152/udp"
-    # ports:
-    #   - "2152:2152/udp"
+    ports:
+      - "2152:2152/udp"
     networks:
       default:
         ipv4_address: ${SGWU_IP}
@@ -127,7 +127,6 @@
       - "5868/sctp"
       - "8805/udp"
       - "2123/udp"
-      - "7777/tcp"
       - "9091/tcp"
     networks:
       default:
@@ -187,8 +186,8 @@
       - "36412/sctp"
       - "2123/udp"
       - "9091/tcp"
-    # ports:
-    #   - "36412:36412/sctp"
+    ports:
+      - "36412:36412/sctp"
     networks:
       default:
         ipv4_address: ${MME_IP}
```

**mme/mme.yaml)**

Change the following `tac` according to the TAC of eNodeB.
```
mme:
...
    sgsap:
      addr: OSMOMSC_IP
      map:
        tai:
          plmn_id:
            mcc: MCC
            mnc: MNC
-->       tac: 1
...
    tai:
      plmn_id:
        mcc: MCC
        mnc: MNC
-->   tac: 1
...
```

<h3 id="build">Build docker image for Open5GS and Kamailio</h3>

Please install the following first.
- [docker-ce](https://docs.docker.com/install/linux/docker-ce/ubuntu)

See [docker_open5gs](https://github.com/herlesupreeth/docker_open5gs) for build instructions.
```
# cd docker_open5gs/base
# docker build --no-cache --force-rm -t docker_open5gs .
...
# cd ../ims_base
# docker build --no-cache --force-rm -t docker_kamailio .
...
```

<h3 id="run">Run docker-compose</h3>

For example, I prepare terminals for each of the following NF groups and execute them in order while checking the startup.

*terminal#1*
```
# set -a
# source .env
# docker compose -f 4g-volte-deploy.yaml up mongo mysql dns webui
```
*terminal#2*
```
# set -a
# source .env
# docker compose -f 4g-volte-deploy.yaml up osmohlr osmomsc
```
*terminal#3*
```
# set -a
# source .env
# docker compose -f 4g-volte-deploy.yaml up hss mme pcrf sgwc sgwu smf upf
```
*terminal#4*
```
# set -a
# source .env
# docker compose -f 4g-volte-deploy.yaml up rtpengine
```
*terminal#5*
```
# set -a
# source .env
# docker compose -f 4g-volte-deploy.yaml up icscf scscf pcscf smsc
```

<h3 id="register_open5gs">Register subscribers information with Open5GS</h3>

**Please also register MSISDN.** At that time, set the APN setting information as follows.
| APN | Type | QCI | ARP | Capability | Vulnerablility | MBR DL/UL(Kbps) | GBR DL/UL(Kbps) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| internet | IPv4 | 9 | 8 | Disabled | Disabled | unlimited/unlimited | |
| ims | IPv4 | 5 | 1 | Disabled | Disabled | 3850/1530 | |
| | | 1 | 2 | Enabled | Enabled | 128/128 | 128/128 |

See below for details.

`18. Install Open5GS in the same machine as Kamailio IMS - Install Open5GS from source`  
https://open5gs.org/open5gs/docs/tutorial/02-VoLTE-setup/

```
http://192.168.0.20:3000/

user: admin
password: 1423
```

<h3 id="register_osmocom">Register the IMSI and MSISDN with OsmoHLR</h3>

Please login to the `osmohlr` container and register by referring to the following.

`6.1 Example: Add/Update/Delete Subscriber via VTY`  
https://downloads.osmocom.org/docs/latest/osmohlr-usermanual.pdf

The following is an example of registering subscriber information for IMSI=001010000001001 and MSISDN=1001.

First, login to the `osmohlr` container.
```
# docker exec -it osmohlr /bin/bash
```
Then telnet to localhost.
```
# telnet localhost 4258
...
OsmoHLR> enable
OsmoHLR#
```
Next, register the subscriber information for IMSI=001010000001001 and MSISDN=1001.
```
OsmoHLR# subscriber imsi 001010000001001 create
% Created subscriber 001010000001001
    ID: 1
    IMSI: 001010000001001
    MSISDN: none
OsmoHLR# subscriber imsi 001010000001001 update msisdn 1001
% Updated subscriber IMSI='001010000001001' to MSISDN='1001'
OsmoHLR#
```
Make sure this subscriber information is registered.
```
OsmoHLR# show subscribers all
ID     MSISDN        IMSI              IMEI              NAM
-----  ------------  ----------------  ----------------  -----
1      1001          001010000001001    -------------    CSPS  
 Subscribers Shown: 1
OsmoHLR#
```
This setting is required to function as **SMS over SGs**.

<h3 id="try">Try VoLTE and SMS</h3>

Make sure that you can make a VoLTE call and SMS to the MSISDN. If your device does not support **SMS over IMS**, you can send SMS with **SMS over SGs** depending on your device.

**Note. Kamailio's SMS (SMS over IMS) doesn't seem to handle multibyte messages properly, which causes garbled characters in SMS.
On the other hand, OsmoMSC (SMS over SGs) seems to handle multibyte messages properly without garbled characters.**

<h4 id="osmomsc_send_command">Send SMS from OsmoMSC VTY terminal (SMS over SGs)</h4>

You can send SMS to the destination terminal by command operation on the OsmoMSC VTY terminal (SMS over SGs).
Please login to the `osmomsc` container and send SMS from the command line as following.

`14 Configuring the Core Network`  
`11.3.3 The list command`  
https://downloads.osmocom.org/docs/latest/osmomsc-usermanual.pdf

**For example, if the following IMSI and MSISDN are registered in OsmoHLR)**
| IMSI | MSISDN | SIM |
| --- | --- | --- |
| 001010000001000 | 1000 | x |
| 001010000001001 | 1001 | o |
| 001010000001002 | 1002 | o |

First, login to the `osmomsc` container.
```
# docker exec -it osmomsc /bin/bash
```
Then telnet to localhost.
```
# telnet localhost 4254
...
OsmoMSC> enable
OsmoMSC#
```

- Command line to send SMS from MSISDN=1001 to MSISDN=1002 where the corresponding SIM exists
```
OsmoMSC# subscriber msisdn 1002 sms sender msisdn 1001 send TEST MESSAGE
```
- Command line to send SMS from MSISDN=1000 to MSISDN=1002 for which there is no corresponding SIM
```
OsmoMSC# subscriber msisdn 1002 sms sender msisdn 1000 send TEST MESSAGE
```

<h2 id="rp4">VoLTE and SMS with Raspberry Pi 4B</h2>

When you try VoLTE and SMS with Raspberry Pi 4B,
as shown [here](https://github.com/s5uishida/install_mongodb_on_ubuntu_for_rp4b), there is a limit to the version of MongoDB that can be installed on the Raspberry Pi 4B.
So make the necessary changes on docker_open5gs for this.

<h3 id="rp4_mongodb">Change how to create the docker image and container of MongoDB</h3>

First, clone this repository.
```
# git clone https://github.com/s5uishida/docker_open5gs_volte_sms_config
```
Copy the `mongo_for_rp4` directory from this repository as the `mongo` directory in the `open5gs_hss_cx` branch of docker_open5gs.
```
# cp -pr docker_open5gs_volte_sms_config/mongo_for_rp4 docker_open5gs/mongo
```
Then, for `4g-volte-deploy.yaml`, in addition to [this change](https://github.com/s5uishida/docker_open5gs_volte_sms_config#changes-in-configuration-files-of-docker_open5gs), change as follows.
```diff
--- 4g-volte-deploy.yaml.orig   2023-08-07 10:58:37.018266553 +0000
+++ 4g-volte-deploy.yaml        2023-08-07 11:18:47.865735359 +0000
@@ -1,14 +1,14 @@
 version: '3'
 services:
   mongo:
-    image: mongo:6.0
+    build: ./mongo
+    image: docker_mongo
     container_name: mongo
-    command: --bind_ip 0.0.0.0
+    command: /bin/sh -c 'mongod --dbpath /data/db --logpath /data/log/mongodb.log --bind_ip 0.0.0.0'
     env_file:
       - .env
     volumes:
-      - mongodbdata:/data/db
-      - mongodbdata:/data/configdb
+      - ./mongo/data:/data
       - /etc/timezone:/etc/timezone:ro
       - /etc/localtime:/etc/localtime:ro
     expose:
@@ -434,5 +433,4 @@
       config:
         - subnet: ${TEST_NETWORK}
 volumes:
-  mongodbdata: {}
   dbdata: {}
```
After this, the configuration and build are the same as above.
I haven't confirmed the operation, but it is probably able to use VoLTE and SMS with Raspberry Pi 4B.

---

[docker_open5gs](https://github.com/herlesupreeth/docker_open5gs) is a excellent software to try **VoLTE** and **SMS** easily. I would like to thank all the software developers and contributors related.

<h2 id="trouble">Trouble Information</h2>

As of 2023.08.09, VoLTE did not work with `open5gs_hss_cx` branch of latest docker_open5gs in my environment.
By reverting the following version, VoLTE was able to work fine.

- docker_open5gs (`commit:bd2501bc11ee344588b1079f21b119f4d3656a7d`) on 2023.03.01
- Open5GS v2.6.1 (`commit:30e420b7a88c2e4d4917b51300f45a1e1632d692`) on 2023.03.08

It may only occur in my environment. In any case, I would like to resolve this issue.

<h2 id="changelog">Changelog (summary)</h2>

- [2023.08.09] Added the trouble information in my environment.
- [2023.08.07] Changed the settings for only using `open5gs_hss_cx` branch.
- [2023.08.05] Added the settings to use VoLTE and SMS with Raspberry Pi 4B. However, there is no confirmation of operation.
- [2022.04.11] Open5GS HSS Cx interface now supports the settings related to SMSC application server.
- [2022.02.27] Initial release.
