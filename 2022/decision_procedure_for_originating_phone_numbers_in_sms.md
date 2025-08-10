---
description: As indicated in a previous article, submit_sm, a SMPP protocol, has a field for loading the originating phone number. However, another previous article indicated that SMS-SUBMIT, a protocol defined by 3GPP, has no such field. How does a SMSC determine originating phone numbers when transferring a short message within the mobile network? This article reveals the answer to this question by observing the SMS transfer flow in a virtual private mobile network.
---

# Decision Procedure for Originating Phone Numbers in SMS

<time datetime="2022-07-18">Jul 18, 2022</time>

---

As indicated in a previous [article](/2022/analysis_and_reproduction_of_spoofed_sms-deliver.md), submit_sm, a SMPP protocol, has a field for loading the originating phone number. However, another previous [article](/2022/transmission_and_detection_of_silent_sms_in_android.md) indicated that SMS-SUBMIT, a protocol defined by 3GPP, has no such field. How does a SMSC determine originating phone numbers when transferring a short message within the mobile network? This article reveals the answer to this question by observing the SMS transfer flow in a virtual private mobile network.

## Summary

Although the originating phone number is not loaded into SMS-SUBMIT, the sender’s identifier is loaded into the initial service request. For an SMS transfer via an MSC (Mobile Switching Centre), the MSC intervenes between the MS (Mobile Station) and SMSC to obtain the phone number associated with the identifier from a VLR (Visitor Location Register). The MSC then forwards the phone number obtained and an SMS-SUBMIT to the SMSC. The SMSC determines the forwarded phone number as the originating phone number for the short message.

## Where is the Originating Phone Number

In submitting short messages to SMSC, submit_sm, via the Internet, has a field that can be loaded with the originating phone number, whereas SMS-SUBMIT, via the mobile network, does not. Figure 1 shows the PDU structures of submit_sm and SMS-SUBMIT as defined in SMPP v3.4 and 3GPP TS 23.040, respectively<sup id="f1">[¹](#fn1)</sup> <sup id="f2">[²](#fn2)</sup>. Submit_sm has a destination_address field for loading the destination phone number as well as a source_addr field for loading the source phone number. In contrast, SMS-SUBMIT has a TP-DA (TP-Destination-Address) field for loading the destination phone number; however, it has no field similar to a TP-OA (TP-Originating-Address) for loading the originating phone number. This means that SMS-SUBMIT does not submit the originating phone number to the SMSC.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure1.webp" width="770" height="427" decoding="async" alt="" /><figcaption>Figure 1: PDU structures of submit_sm and SMS-SUBMIT.</figcaption></figure>

So why does a mobile device display the originating phone number of the short message received over the mobile network? In other words, how does the SMSC determine the originating phone number to load into the TP-OA field in the SMS-DELIVER? To answer these questions, I first review the procedure for determining the originating phone number based on 3GPP specifications. Next, I build a virtual mobile network using Osmocom and analyze the network traffic and logs when short messages are transferred. After clarifying the procedure for determining the originating phone number, I present the answers and remaining tasks.

## How the Decision Procedure Works

According to SMS technical specification 3GPP TS 23.040, the SMS-SUBMIT sent from a mobile device travels through several entities before being received by the SMSC. Figure 2 shows the entities involved in an SMS-SUBMIT transfer based on Section 10.2. The SMS-SUBMIT sent from the MS (synonymous with “mobile device” in this article) is first received by the MSC or SGSN (Serving GPRS Support Node). The MSC is an exchange that performs switching functions for MSs, whereas the SGSN performs packet switching functions for MSs and is used instead of the MSC for SMS transfers over the GPRS (General Packet Radio Service). The SMS-SUBMIT sent to the MSC or SGSN is then received by the SMS-IWMSC (Short Message Service Interworking MSC). The SMS-IWMSC provides the capability of receiving a short message from within the PLMN (Public Land Mobile Network) and submitting it to the SMSC. Therefore, the SMS-SUBMIT transfer involves the intervention of the MSC or SGSN in addition to the SMS-IWMSC.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure2.webp" width="770" height="345" decoding="async" alt="" /><figcaption>Figure 2: Entities involved in the SMS-SUBMIT transfer procedure.</figcaption></figure>

Upon receiving the SMS-SUBMIT, the MSC obtains the originating phone number from the VLR. Section 8.2.1 of the specification describes the behavior of the MSC when it receives a TPDU (synonymous with “SMS-SUBMIT” in this context) from the MS as follows:

> retrieving information from the VLR ("sendInfoForMO-SMS", see clause 10); the MSISDN of the MS and, when appropriate, error information.

The VLR is a database of information about MSs connected to the network managed by the MSC, and it stores the subscriber’s ID, i.e., the IMSI (International Mobile Subscriber Identity), and phone number, i.e., the MSISDN (Mobile Station ISDN Number). In addition, Section 10.2 describes the response of the VLR to requests from the MSC as follows:

> A successful VLR response carries the MSIsdn of the originating MS being transferred to the SC at SM-RL.

From these descriptions, the MSC evidently determines the originating phone number of the SMS-SUBMIT based on the information stored in the VLR. However, how the SGSN determines the originating phone number is not described in Sections 8.2.2, 8.2.3, or 10.2. Therefore, this article focuses on the MSC procedure for determining the originating phone number and attempts to reproduce it.

## Reproduction of SMS

To reproduce an SMS transfer via the MSC, I built a virtual mobile network using software provided by Osmocom. Osmocom is an open-source mobile communications enabling project that provides a variety of software for building 2G and 3G mobile networks<sup id="f3">[³](#fn3)</sup>. I conducted the test in a 2G circuit-switched network built by combining their software<sup id="f4">[⁴](#fn4)</sup>. Figure 3 illustrates the network structure. OsmoMSC, which is Osmocom’s implementation of an MSC, has a built-in SMSC and VLR, whereas SMS-IWMSC does not. OsmoHLR is responsible for the HLR (Home Location Register), which stores the source of subscriber information that is cached by the VLR. Between OsmoMSC and the MS, OsmoSTP acts as a signal transfer point, OsmoBTS as a base transceiver station, and OsmoBSC as the controller of OsmoBTS. In addition, the use of OsmocomBB and virtphy as the MS enables software-only testing<sup id="f5">[⁵](#fn5)</sup>.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure3.webp" width="770" height="421" decoding="async" alt="" /><figcaption>Figure 3: Short message transfer in the Osmocom network.</figcaption></figure>

The test reproduced the sending and receiving of short messages using two virtual MSs connected to the mobile network. OsmocomBB can launch multiple MSs and provides an example config file required for this<sup id="f6">[⁶](#fn6)</sup>. In this test, the SIM setting was changed from `sim reader` to `sim test` to use a virtual MS, and the SMSC setting was changed from `no sms-service-center` to `sms-service-center 447785016005` to send the SMS. The number `447785016005` is the SMSC address hard-coded into OsmoMSC<sup id="f7">[⁷](#fn7)</sup>. Before launching the MSs, their SIM information and dummy phone numbers listed in the config file were registered to OsmoHLR, as shown in Figure 4. The OsmocomBB was launched after two instances of virtphy were activated with the layer 2 sockets specified in the config file. Figure 5 shown the sending and receiving of a short message at the OsmocomBB terminal. A short message was sent and received by the OsmocomBB connected through the terminal on the left; additionally, it was forwarded by the OsmoMSC connected through the terminal on the right. Network traffic was captured by the tcpdump executed from the bottom terminal. In Figure 5, I checked the subscriber information cached in the VLR of the OsmoMSC before sending a short message. After sending a short message from MS1 to MS2, when received by MS2, MS2 shows the originating phone number, which is MS1 number `818001234567` in this case. By analyzing the network traffic and logs generated by this test, I could identify the procedure for determining the originating phone number.

```
OsmoHLR# subscriber imsi 001010000000001 create
% Created subscriber 001010000000001
    ID: 1
    IMSI: 001010000000001
    MSISDN: none
OsmoHLR# subscriber id 1 update msisdn 818001234567
% Updated subscriber IMSI='001010000000001' to MSISDN='818001234567'
OsmoHLR# subscriber id 1 update aud2g comp128v1 ki 00000000000000000000000000000000

OsmoHLR# subscriber imsi 001010000000002 create
% Created subscriber 001010000000002
    ID: 2
    IMSI: 001010000000002
    MSISDN: none
OsmoHLR# subscriber id 2 update msisdn 819001234567
% Updated subscriber IMSI='001010000000002' to MSISDN='819001234567'
OsmoHLR# subscriber id 2 update aud2g comp128v1 ki ffffffffffffffffffffffffffffffff
```
<figure><figcaption>Figure 4: Subscriber information registered in OsmoHLR.</figcaption></figure>

<figure><video controls muted playsinline poster="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure5.webp" src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure5.mp4" type="video/mp4"></video><figcaption>Figure 5: Sending and receiving a short message using OsmocomBB.</figcaption></figure>

## Clarification of the Decision Procedure

First, I analyzed the network traffic captured when the MS connected to the mobile network. When the MS connects, the MSC caches  the subscriber information from the HLR to the VLR and assigns a TMSI (Temporary Mobile Subscriber Identity) to the MS. Figure 6 shows the DTAP (Direct Transfer Application Part) and GSUP (Generic Subscriber Update Protocol) packets captured when OsmocomBB launched. DTAP is a protocol designed for the A-interface between the BSC and MSC. It supports transparent message transfers between an MS and MSC. Moreover, the GSUP is an Osmocom-specific protocol used between OsmoMSC and OsmoHLR and is an alternative to the MAP (Mobile Application Part) protocol<sup id="f8">[⁸](#fn8)</sup>. Because the two MSs run in OsmocomBB, the request and response for both protocols are captured in duplicate. Figure 7 presents a sequence diagram of the traffic in important packets. Observing the traffic flow, the MS first sends its IMSI to the OsmoMSC via a `Location Updating Request` (Figure 8). Next, the OsmoMSC sends the IMSI received to the OsmoHLR via an `UpdateLocation Request` (Figure 9), and the OsmoHLR sends the subscriber information associated with the received IMSI to the OsmoMSC via an `InsertSubscriberData Request` (Figure 10). In the case of Figure 10, MS1’s subscriber information includes the MSISDN `818001234567`. Subsequently, the OsmoMSC issues a TMSI associated with the IMSI and sends it to the MS via a `Location Updating Accept` (Figure 11). This flow enables the VLR of OsmoMSC to cache the IMSI, MSISDN, and TMSI associated with the MS when the MS connects to the network. The information cached in the VLR can be checked using the `show subscriber cache` command, as shown in Figure 5.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure6.webp" width="770" height="267" decoding="async" alt="" /><figcaption>Figure 6: Packets captured when OsmocomBB was launched.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure7.webp" width="770" height="255" decoding="async" alt="" /><figcaption>Figure 7: Sequence diagram of TMSI allocation.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure8.webp" width="770" height="243" decoding="async" alt="" /><figcaption>Figure 8: Packet details of Location Updating Request.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure9.webp" width="770" height="172" decoding="async" alt="" /><figcaption>Figure 9: Packet details of UpdateLocation Request.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure10.webp" width="770" height="186" decoding="async" alt="" /><figcaption>Figure 10: Packet details of InsertSubscriberData Request.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure11.webp" width="770" height="202" decoding="async" alt="" /><figcaption>Figure 11: Packet details of Location Updating Accept.</figcaption></figure>

Next, I analyzed the network traffic captured when a short message was sent or received. Before submitting the short message, sender MS1 submitted its TMSI to the MSC. Figure 12 shows the DTAP packets captured when a short message is sent and received, and Figure 13 provides a sequence diagram of the traffic in important packets. Observing  the traffic flow,  MS1 first sends its TMSI to the OsmoMSC via a `CM Service Request` (Figure 14), and then sends an SMS-SUBMIT to the OsmoMSC (Figure 15). Subsequently, receiver MS2 sends its TMSI to the OsmoMSC via a `Paging Response` (Figure 16). Although not shown in Figure 12, immediately prior to this, the OsmoMSC sends a `PAGING CoMmanD` loaded with MS2’s TMSI over the RSL (Radio Signalling Link) protocol (Figure 17). After the paging response, MS2 receives an SMS-DELIVER from the OsmoMSC (Figure 18). As shown in Figure 18, MS1’s phone number `818001234567` was loaded into the TP-OA field of the SMS-DELIVER. The first half of the flow enables the MSC to obtain the MSISDN associated with the sender’s TMSI from the VLR upon receiving the SMS-SUBMIT.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure12.webp" width="770" height="229" decoding="async" alt="" /><figcaption>Figure 12: Packets captured when transferring the short message.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure13.webp" width="770" height="317" decoding="async" alt="" /><figcaption>Figure 13: Sequence diagram of SMS transfer.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure14.webp" width="770" height="230" decoding="async" alt="" /><figcaption>Figure 14: Packet details of CM Service Request.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure15.webp" width="770" height="329" decoding="async" alt="" /><figcaption>Figure 15: Packet details of SMS-SUBMIT.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure16.webp" width="770" height="217" decoding="async" alt="" /><figcaption>Figure 16: Packet details of Paging Response.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure17.webp" width="770" height="310" decoding="async" alt="" /><figcaption>Figure 17: Packet details of PAGING CoMmanD.</figcaption></figure>

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure18.webp" width="770" height="329" decoding="async" alt="" /><figcaption>Figure 18: Packet details of SMS-DELIVER.</figcaption></figure>

Subsequently, I analyzed the log output when the OsmoMSC received the SMS-SUBMIT. Upon receiving the SMS-SUBMIT, the MSC obtains the sender’s MSISDN from the VLR and can forward it to the SMSC. The VLR and SMSC are built into OsmoMSC; thus, no traffic exists between them and the MSC. Therefore, I inferred the behavior of the OsmoMSC after receiving the SMS-SUBMIT based on its output logs. Figure 19 shows a part of the debug log output when the OsmoMSC received the SMS-SUBMIT. Log 1 shows that MS1’s TMSI was received by the `CM Service Request`. Log 2 indicates that the MSISDN `818001234567` associated with MS1’s TMSI was obtained from the VLR, and log 3 shows that the short message transferred by SMS-SUBMIT and MS1’s MSISDN are stored in the database. Based on these behaviors, I infer that the SMSC database stores a short message according to its originating phone number.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure19.webp" width="770" height="433" decoding="async" alt="" /><figcaption>Figure 19: Part of the debug log output of the OsmoMSC.</figcaption></figure>

Finally, I confirmed that the actual data is stored in the database when the SMSC receives the SMS-SUBMIT. Within the SMSC database, a short message is stored according to its originating phone number. OsmoMSC was designed to temporarily store short messages received in an SQLite database called `sms.db`. Figure 20 shows the data temporarily stored in the database when a short message is resent from MS1 to MS2. The short messages sent from MS1 and MS1’s MSISDN `818001234567` can be seen to be stored in the same record. Therefore, the SMSC can load the originating phone number into the TP-OA field by building the SMS-DELIVER based on this record.

<figure><img src="/assets/2022/decision_procedure_for_originating_phone_numbers_in_sms/25_figure20.webp" width="770" height="139" decoding="async" alt="" /><figcaption>Figure 20: Short message stored in the SMSC database.</figcaption></figure>

## Conclusion and Future Work

By observing the SMS transfer flow via an MSC in a mobile network, I concluded that the originating phone number of a short message is determined based on the information stored in the VLR. For an SMS transferred by an MSC, the MSC intervenes between the MS and SMSC by referring to the MS information stored in the VLR. The MSC obtains the phone number associated with the identifier submitted by the MS from the VLR before sending the SMS-SUBMIT. The MSC forwards the phone number obtained and SMS-SUBMIT to the SMSC, which can then load the originating phone number into the TP-OA field in the SMS-DELIVER.

However, modern SMS transfer technology is not limited to MSCs. The answers in this article are based on SMS transfers performed by MSCs in a 2G network. Therefore, it remains unclear how the originating phone number is determined in SMS transfer via an SGSN. Furthermore, as modern mobile networks evolve to 3G, 4G, and 5G, SMS transfer protocols evolve accordingly and diversely<sup id="f9">[⁹](#fn9)</sup>. Future work should reveal the procedure for determining the originating phone numbers in other SMS transfer protocols.

---

<sup id="fn1">[¹](#f1)</sup> [Short Message Peer to Peer Protocol Specification v3.4 - SMPP.org](https://smpp.org/SMPP_v3_4_Issue1_2.pdf)  
<sup id="fn2">[²](#f2)</sup> [3GPP TS 23.040, Technical realization of the Short Message Service (SMS)](https://www.3gpp.org/DynaReport/23040.htm)  
<sup id="fn3">[³](#f3)</sup> [Open Source Mobile Communications](https://osmocom.org/)  
<sup id="fn4">[⁴](#f4)</sup> [Part of this Complete Network - Cellular Network Infrastructure - Osmocom](https://osmocom.org/projects/cellular-infrastructure/wiki/Osmocom_Network_In_The_Box#Part-of-this-Complete-Network)  
<sup id="fn5">[⁵](#f5)</sup> [Virtual Um - Cellular Network Infrastructure - Osmocom](https://osmocom.org/projects/cellular-infrastructure/wiki/Virtual_Um)  
<sup id="fn6">[⁶](#f6)</sup> [osmocom-bb/multi_ms.cfg - osmocom-bb - Osmocom gitea](https://gitea.osmocom.org/phone-side/osmocom-bb/src/commit/785450c4bfdbcde4923b60bcb25b6367c74bb961/doc/examples/mobile/multi_ms.cfg)  
<sup id="fn7">[⁷](#f7)</sup> [osmo-msc/gsm_04_11.c#L1180 - osmo-msc - Osmocom gitea](https://gitea.osmocom.org/cellular-infrastructure/osmo-msc/src/tag/1.8.0/src/libmsc/gsm_04_11.c#L1180)  
<sup id="fn8">[⁸](#f8)</sup> [GSUP - Cellular Network Infrastructure - Osmocom](https://osmocom.org/projects/cellular-infrastructure/wiki/GSUP)  
<sup id="fn9">[⁹](#f9)</sup> [SMS Evolution Version 2.0 - GSMA](https://www.gsma.com/newsroom/wp-content/uploads/NG.111-v2.0.pdf)
