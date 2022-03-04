# VoLTE and SMS Configuration for docker_open5gs
This briefly describes the settings for using **VoLTE** and **SMS** of [docker_open5gs](https://github.com/herlesupreeth/docker_open5gs).

---
<h2 id="toc">Table of Contents</h2>

- [Overview of Network Configuration](#overview)
- [Use the master branch of docker_open5gs](#branch_master)
  - [Changes in configuration files of docker_open5gs](#change_config)
  - [Build docker image for Open5GS and Kamailio](#build)
  - [Run docker-compose](#run)
  - [Register subscribers information with Open5GS](#register_open5gs)
  - [Register the IMSI and MSISDN with OsmoHLR](#register_osmocom)
  - [Register IMS subscription with FHoSS](#register_fhoss)
  - [Try VoLTE and SMS](#try)
    - [Send SMS from OsmoMSC VTY terminal (SMS over SGs)](#osmomsc_send_command)
- [Use the open5gs_hss_cx branch of docker_open5gs](#branch_open5gs)
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
For eNodeB, set static routing to eNodeB for packets going from eNodeB to the Docker host network (`172.22.0.0/24`).
```
# ip route add 172.22.0.0/24 via 192.168.0.20
```
If eNodeB does not have a static routing setting function, build a network so that the Docker host becomes the default GW.

<h2 id="branch_master">Use the master branch of docker_open5gs</h2>

In this branch you can try **VoLTE** and **SMS over IMS** / **SMS over SGs**.   
**Note.** For **SMS over SGs** of docker_open5gs, 2G/3G radio access is not used at all rather it uses only the core network (MSC + SMSC + HLR).
```
# git clone https://github.com/herlesupreeth/docker_open5gs
# cd docker_open5gs
```

<h3 id="change_config">Changes in configuration files of docker_open5gs</h3>

**.env)**

| Item | Value |
| --- | --- |
| MCC | 001 |
| MNC | 01 |
| DOCKER_HOST_IP | 192.168.0.20 |
| SGWU_ADVERTISE_IP | 192.168.0.20 |

**docker-compose.yaml)**

Publish the ports of each of the following NFs.
```
  sgwu:
...
-    # ports:
-    #   - "2152:2152/udp"
+    ports:
+      - "2152:2152/udp"
...
  mme:
...
-    # ports:
-    #   - "36412:36412/sctp"
+    ports:
+      - "36412:36412/sctp"
...
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
# docker-compose up mongo mysql dns webui
```
*terminal#2*
```
# set -a
# source .env
# docker-compose up osmohlr osmomsc
```
*terminal#3*
```
# set -a
# source .env
# docker-compose up nrf hss mme pcrf sgwc sgwu smf upf
```
*terminal#4*
```
# set -a
# source .env
# docker-compose up rtpengine fhoss
```
*terminal#5*
```
# set -a
# source .env
# docker-compose up smsc icscf scscf pcscf
```

<h3 id="register_open5gs">Register subscribers information with Open5GS</h3>

Please also register MSISDN. At that time, set the APN setting information as follows.
| APN | Type | QCI | ARP | Capability | Vulnerablility | MBR DL/UL(Kbps) | GBR DL/UL(Kbps) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| internet | IPv4 | 9 | 8 | Disabled | Disabled | unlimited/unlimited | |
| ims | IPv4 | 5 | 1 | Disabled | Disabled | 3850/1530 | |
| | | 1 | 2 | Enabled | Enabled | 128/128 | 128/128 |

See below for details.

`18. Install Open5GS in the same machine as Kamailio IMS - Install Open5GS from source`  
https://open5gs.org/open5gs/docs/tutorial/02-VoLTE-setup/

<h3 id="register_osmocom">Register the IMSI and MSISDN with OsmoHLR</h3>

Please login to the `osmohlr` container and register by referring to the following.

`6.1 Example: Add/Update/Delete Subscriber via VTY`  
https://ftp.osmocom.org/docs/latest/osmohlr-usermanual.pdf

This setting is required to function as **SMS over SGs**.

<h3 id="register_fhoss">Register IMS subscription with FHoSS</h3>

Please refer to the following to register.

`20. Add IMS subscription use in FoHSS as follows from the Web GUI`  
https://open5gs.org/open5gs/docs/tutorial/02-VoLTE-setup/

```
http://192.168.0.20:8080/hss.web.console/
```
| Item | Value |
| --- | --- |
| user | hssAdmin |
| password | hss |

Then, for each IMPU, you need to select `smsc_sp` for Service profile rather than `default_sp`.

<h3 id="try">Try VoLTE and SMS</h3>

Make sure that you can make a VoLTE call and SMS to the MSISDN. If your device does not support **SMS over IMS**, you can send SMS with **SMS over SGs** depending on your device.

**Note. Kamailio seems to have a bug in handling multibyte messages, which causes garbled characters in SMS.**

<h4 id="osmomsc_send_command">Send SMS from OsmoMSC VTY terminal (SMS over SGs)</h4>

You can send SMS to the destination terminal by command operation on the OsmoMSC VTY terminal (SMS over SGs).
Please login to the `osmomsc` container and send SMS from the command line as following.

`11.3.3 The list command`  
https://ftp.osmocom.org/docs/latest/osmomsc-usermanual.pdf

**For example, if the following IMSI and MSISDN are registered in osmohlr)**
| IMSI | MSISDN | SIM |
| --- | --- | --- |
| 001010000001000 | 1000 | x |
| 001010000001001 | 1001 | o |
| 001010000001002 | 1002 | o |

- Command line to send SMS from MSISDN=1001 to MSISDN=1002 where the corresponding SIM exists
```
OsmoMSC# subscriber msisdn 1002 sms sender msisdn 1001 send TEST MESSAGE
```
- Command line to send SMS from MSISDN=1000 to MSISDN=1002 for which there is no corresponding SIM
```
OsmoMSC# subscriber msisdn 1002 sms sender msisdn 1000 send TEST MESSAGE
```

<h2 id="branch_open5gs">Use the open5gs_hss_cx branch of docker_open5gs</h2>

In this branch you can try **VoLTE** and **SMS over SGs**.
Kamailio's S-CSCF and I-CSCF can communicate with Open5GS HSS(Cx) instead of FHoSS. However, since Open5GS HSS(Cx) has no settings related to the SMSC application server, S-CSCF does not have the information to forward SIP messages to SMSC(IMS), and so **SMS over IMS** cannot be used. In this case, if your device supports CSFB, send SMS with **SMS over SGs** using OsmoHLR and OsmoMSC.
```
# git clone https://github.com/herlesupreeth/docker_open5gs
# cd docker_open5gs
# git checkout open5gs_hss_cx
...
```

---

[docker_open5gs](https://github.com/herlesupreeth/docker_open5gs) is a excellent software to try **VoLTE** and **SMS** easily. I would like to thank all the software developers and contributors related.
