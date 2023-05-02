---
description: A previous article clarified the decision procedure for originating numbers in SMS via a Mobile Switching Center (MSC). An MSC is a 2G circuit-switched network node, whereas modern 4G and 5G networks can provide SMS over IP Multimedia Subsystem (IMS) connected to a packet-switched network. By reviewing the technical specifications and analyzing packets captured in a private mobile network, this article clarifies the answer to how Short Message Service Centers (SMSCs) determine the originating number in SMS over IMS.
---

# Decision Procedure for Originating Numbers in SMS over IMS

<p class="modest" align="left">Mar 27, 2023</p>

---

A previous [article](/2022/decision_procedure_for_originating_phone_numbers_in_sms.md) clarified the decision procedure for originating numbers in SMS via a Mobile Switching Center (MSC). An MSC is a 2G circuit-switched network node, whereas modern 4G and 5G networks can provide SMS over IP Multimedia Subsystem (IMS) connected to a packet-switched network. By reviewing the technical specifications and analyzing packets captured in a private mobile network, this article clarifies the answer to how Short Message Service Centers (SMSCs) determine the originating number in SMS over IMS.

## Summary

In SMS over IMS, the originating number is determined based on the P-Asserted-Identity header of the Session Initiation Protocol (SIP) request. The SMS Protocol Data Units (PDUs) are encapsulated in the SIP and forwarded to the IMS. Therefore, the SMSC receives a SIP MESSAGE request encapsulating an SMS-SUBMIT from a user equipment (UE). The P-Asserted-Identity header of this request is set to the public user identity containing the UE phone number. The 3GPP technical specifications expect SMSCs to determine the originating number from this header, and the SMSC tested in the private mobile network met these expectations.

## Introduction to SMS over IMS

An IMS can provide Internet Protocol (IP)-based multimedia services via mobile networks. The 2G and 3G mobile networks were divided into circuit-switched networks that carried voice calls and packet-switched networks that carried data sessions. However, on and after 4G, the circuit-switched network was eliminated and All-IP mobile networks consisting only of packet-switched networks existed. The IMS was standardized by the 3GPP in collaboration with the IETF to implement traditional services provided over circuit-switched networks in packet-switched networks using Internet technology<sup id="f1">[¹](#fn1)</sup>. Consequently, an IMS can provide services such as voice calling and short messaging without relying on circuit-switched networks.

Although traditional SMS is provided over circuit-switched networks, IMS can provide it over packet-switched networks. The technical specifications for SMS over IMS are defined in 3GPP TS 24.341<sup id="f2">[²](#fn2)</sup> for the protocol details and 3GPP TS 23.204<sup id="f3">[³](#fn3)</sup> for the required capabilities and enhancements. Figure 1 shows the architecture for providing an SMS based on Section 5.1 of 3GPP TS 23.204. The Call Session Control Function (CSCF) is the IMS node that controls the SIP packets. SIP plays the role of a session control protocol for the IMS. Therefore, in the SMS over IMS, SMS PDUs are encapsulated and forwarded by the SIP. Furthermore, the IP-Short-Message-Gateway (IP-SM-GW) intermediates between the Serving-CSCF (S-CSCF) and SMSC.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure1.png" />
<p class="modest" align="center">Figure 1: SMS over IMS architecture.</p>

## Identification of the Sender

The IP-SM-GW is an application server (AS) that supports interworking with traditional SMS and identifies the sender of short messages. Figure 2 shows the sequence of short message submission in SMS over IMS, based on the 3GPP TS 23.204 clause 6.3 and TS 24.341 clause B.5. The SMS-SUBMIT encapsulated in a SIP MESSAGE request sent by the UE is received by the S-CSCF via the Proxy-CSCF (P-CSCF) and then forwarded to the IP-SM-GW according to the initial filter criteria (iFC), which is the trigger set for service provision. The IP-SM-GW authorizes service usage based on subscriber information obtained from the Home Subscriber Server (HSS). Specifically, the IP-SM-GW verifies that the sender of the SMS-SUBMIT is authorized to use the SMS and then forwards it to the SMSC.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure2.png" />
<p class="modest" align="center">Figure 2: Sequence diagram of short message submission in SMS over IMS.</p>

IP-SM-GW identifies the sender based on the public user identity contained in the MESSAGE request. To identify subscribers to an IMS, mobile network operators assign a private user identity and one or more public user identities to each subscriber. Public user identities are primarily used as contact information, whereas private user identities are used for subscriber identification and authentication. Therefore, a private user identity performs a similar function in the IMS as an IMSI (International Mobile Subscriber Identifier), and a public user identity performs a similar function in the IMS as an MSISDN (Mobile Subscriber ISDN Number)<sup id="f4">[⁴](#fn4)</sup>. Private user identities are in the form of a Network Access Identifier (NAI), whereas public user identities are in the form of a SIP URI or TEL URI. IP-SM-GW can identify the sender based on these values stored in the UE’s Subscriber Identity Module (SIM) and the operator’s HSS.

The originating number of short messages is submitted by the header fields of the MESSAGE request. Section 5.3.1.2 of 3GPP TS 24.341 specifies that the sender shall send a MESSAGE request with the following information:

> When an SM-over-IP sender wants to submit an SM over IP, the SM-over-IP sender shall send a SIP MESSAGE request with the following information:
>
> b) the From header, which shall contain a public user identity of the SM-over-IP sender;

Because the format of the public user identity is a SIP URI or TEL URI, the From header may contain the originating number. In addition, the section notes the following regarding the place of the originating number:

> NOTE 2: The IP-SM-GW will have to use an address of the SM-over-IP sender that the SC can process (i.e. an E.164 number). This address will come from a tel URI in a P-Asserted-Identity header (as defined in RFC 3325 [13]) placed in the SIP MESSAGE request by the P-CSCF or S-CSCF.

According to RFC 3325, which is a SIP extension, the value of the P-Asserted-Identity header is also in the form of a SIP URI or TEL URI<sup id="f5">[⁵](#fn5)</sup>. Figure 3 shows the MESSAGE request from the S-CSCF to the IP-SM-GW (step 3 in Figure 2), as exemplified in Section B.5 of 3GPP TS 24.341. The second P-Asserted-Identity header in the request contains the phone number `+12125551111`. Thus, it can be inferred that the originating number of the short message is determined based on the From header, the P-Asserted-Identity header of the request, or both.

```
MESSAGE sip:sc.home1.net SIP/2.0
Via: SIP/2.0/UDP scscf1.home1.net;branch=z9hG4bK344a651, SIP/2.0/UDP pcscf1.visited1.net;branch=z9hG4bK240f341, SIP/2.0/UDP [5555::aaa:bbb:ccc:ddd]:1357;comp=sigcomp; branch=z9hG4bKnashds7
Max-Forwards: 68
Route: <sip:ipsmgw.home1.net;lr>, <sip:cb03a0s09a2sdfglkj490333@scscf1.home1.net;lr>
P-Asserted-Identity: "John Doe" <sip:user1_public1@home1.net>
P-Asserted-Identity: tel:+12125551111
From: <sip:user1_public1@home1.net>; tag=171828
To: <sip:sc.home1.net>
Call-ID: cb03a0s09a2sdfglkj490333
Cseq: 666 MESSAGE
Content-Type: application/vnd.3gpp.sms
Content-Length: (…)
```
<p class="modest" align="center">Figure 3: Example of a SIP MESSAGE request from S-CSCF to IP-SM-GW.</p>

## Reproduction of SMS over IMS

To clarify the decision procedure for originating numbers in SMS over IMS, a private mobile network was built to reproduce short message transfers. The mobile network was built using docker_open5gs<sup id="f6">[⁶](#fn6)</sup>. Docker_open5gs is a set of Docker files that combines Open5GS, srsRAN, and Kamailio to build a mobile network, including IMS. Open5GS<sup id="f7">[⁷](#fn7)</sup> is an open-source implementation of 4G and 5G core networks and srsRAN<sup id="f8">[⁸](#fn8)</sup> is an open-source implementation of 4G and 5G radio access networks. Kamailio<sup id="f9">[⁹](#fn9)</sup> is an open-source SIP server that includes an SMSC. Therefore, SMS over IMS can be implemented by running docker_open5gs.

### Deployment of a Private Mobile Network

In this test, SMS over IMS was deployed using a 4G core network. Figure 4 shows the deployed mobile network. For the test, the open5gs_hss_cx branch of docker_open5gs was checked out and deployed as a non-standalone. Therefore, a short message sent by the UE is first received by the eNodeB (eNB). It is then forwarded through the Evolved Packet Core (EPC) network to the IMS. Because docker_open5gs does not include IP-SM-GW, the short message is directly forwarded from an S-CSCF to an SMSC.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure4.png" />
<p class="modest" align="center">Figure 4: Short message transfer in the private mobile network.</p>

To connect the UEs to the network, subscriber information must be registered in both SIM cards and the HSS. Subscriber information was written on the sysmoISIM<sup id="f10">[¹⁰](#fn10)</sup> programmable SIM card using pySim<sup id="f11">[¹¹](#fn11)</sup> and registered to the HSS using the Open5GS WebUI. Figure 5 shows the subscriber information for UE1 written on the SIM card and Figure 6 shows the subscriber information registered to the HSS. SysmoISIM is an IP multimedia Service Identity Module (ISIM) that can store IMS-specific parameters. However, in this test, the private user identity, public user identity, and home network domain URI were not written on it. The MSISDN was registered only in the HSS. The MSISDN `818001234567` in Figure 6 is the phone number for UE1. UE2’s subscriber information is registered in the same way, and `819001234567` is set as the phone number.

```
$ ./pySim-prog.py -p 0 -n Open5GS -i 001010000000001 -s ███████████████████ -a ████████ -x 001 -y 01 -k 00000000000000000000000000000001 -o 11111111111111111111111111111111
Using PC/SC reader interface
Ready for Programming: Insert card now (or CTRL-C to cancel)
Autodetected card type: sysmoISIM-SJA2
Generated card parameters :
 > Name     : Open5GS
 > SMSP     : e1ffffffffffffffffffffffff0581005155f5ffffffffffff000000
 > ICCID    : ███████████████████
 > MCC/MNC  : 001/01
 > IMSI     : 001010000000001
 > Ki       : 00000000000000000000000000000001
 > OPC      : 11111111111111111111111111111111
 ...
```
<p class="modest" align="center">Figure 5: Subscriber information written on sysmoISIM.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure6.png" />
<p class="modest" align="center">Figure 6: Subscriber information registered to Open5GS HSS.</p>

### Demonstration of Short Message Transfer

Short message transfer over the IMS was reproduced between UE1 and UE2. In the test, docker_open5gs was run on Ubuntu running on VMware on a MacBook. The LimeSDR Mini v2.2<sup id="f12">[¹²](#fn12)</sup> was used as the eNB’s radio hardware. A Galaxy S10 was used as UE1, and an iPhone 12 Pro was used as UE2. Before connecting to the network, UE1 and UE2 had sysmoISIM inserted and the APN settings shown in Figure 6 configured. In addition, the IMS settings of UE1 were overridden by CoIMS<sup id="f13">[¹³](#fn13)</sup>.

Figure 7 shows a demonstration of a short message transfer in a shielded room. In the figure, UE1 (right) sends a short message to UE2 (left). On the screen of UE1, the phone number `819001234567` of UE2 is displayed as the destination number of the message. On the screen of UE2, which received the message, the phone number `818001234567` of UE1 is displayed as the originating number. By analyzing the packets captured in the test, the decision procedure for the originating number can be clarified.

<video controls muted playsinline poster="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure7.png" src="https://raw.githubusercontent.com/atsunoda/www.akaki.io/master/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure7.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 7: Demonstration of short message transfer in SMS over IMS.</p>

## Clarification of the Decision Procedure

Before using SMS over IMS, the UE must complete two processes: registration and subscription. The registration process involves the UE registering with the IMS. During this process, the UE is authenticated by the IMS and authorized to use the IMS services and assigned identities. On the other hand, the subscription process involves the UE subscribing to a notification of its own information registered with the IMS. This process allows the UE to be notified of the status of its public user identity. The next section analyzes the packets captured during the registration and subscription processes when UE1 connects to the network. UE2 performs these processes as well. After reviewing these preprocessing steps, the packets during the short message transfer between UE1 and UE2 were analyzed, and finally, the SMSC process was analyzed.

### Preprocess for IMS Services

The original purpose of the registration process is to bind public user identities and contact addresses. Simultaneously, the UE is authenticated by the IMS and is authorized to use the IMS services and assigned identities. Generally, the registration process is completed in two phases: obtaining the challenge required for authentication, and authenticating the UE using this challenge. This article focuses on the authentication phase. Figure 8 shows the SIP and Diameter packets as UE1 completes the authentication phase of the registration process, and Figure 9 shows the sequences.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure8.png" />
<p class="modest" align="center">Figure 8: Packets captured in the authentication phase of the registration process.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure9.png" />
<p class="modest" align="center">Figure 9: Sequence diagram of the authentication phase of the registration process.</p>

The UE submits credentials based on the challenge phase to the S-CSCF for authentication. Figure 10 shows the REGISTER request in Step 1 of Figure 9. The credentials used for authentication are included in the Authorization header. At this time, UE1 has not been assigned a public user identity; thus, the From header is set to a temporary identity based on the IMSI. The Contact header is set to a SIP URI containing the IP address of UE1, plus a `+g.3gpp.smsip` parameter requesting SMS over IMS service. This REGISTER request is forwarded to S-CSCF via P-CSCF and Interrogating-CSCF (I-CSCF).

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure10.png" />
<p class="modest" align="center">Figure 10: Packet details of the REGISTER request sent from UE1.</p>

Upon receiving the REGISTER request, the S-CSCF verifies the credentials of the UE based on the authentication vectors obtained from the HSS during the challenge phase. If the authentication is successful, the S-CSCF sends a Diameter Server-Assignment-Request (SAR) to the HSS and receives a Server-Assignment Answer (SAA) with the user profile of the newly registered UE. Figure 11 shows the user profile obtained in Step 7 of Figure 9. The user profile in the XML format contains two public user identities `sip:818001234567@ims.mnc001.mcc001.3gppnetwork.org` and `tel:818001234567` newly assigned to UE1. The user profile also contains an iFC. This instructs the S-CSCF to forward it to the SMSC if the Content-Type header of the received MESSAGE request is `application/vnd.3gpp.sms`. The two public user identities assigned to UE1 are notified to UE1 via the P-Associated-URI header of the 200 OK response (Figure 12 and Step 10 of Figure 9).

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure11.png" />
<p class="modest" align="center">Figure 11: Details of the user profile on the Diameter SAA packet.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure12.png" />
<p class="modest" align="center">Figure 12: Packet detail of the 200 OK response notifying public user identities.</p>

After completing the registration process for the IMS, the UE performs the subscription process to be notified of its own registration status, as maintained by the S-CSCF. This process allows the UE to be notified of whether its own public user identities are active and what contact addresses are bound to public user identities. Figure 13 shows the SIP packets as UE1 performed the subscription process, and Figure 14 shows the sequences. The UE submits its own public user identity to the S-CSCF at the beginning of the subscription process. Figure 15 shows the SUBSCRIBE request in Step 1 of Figure 14. In the request, it can be seen that UE1’s public user identity is set in the Request-URI and P-Preferred-Identity headers. This request is forwarded to S-CSCF via P-CSCF.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure13.png" />
<p class="modest" align="center">Figure 13: Packets captured in the subscription process.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure14.png" />
<p class="modest" align="center">Figure 14: Sequence diagram of the subscription process.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure15.png" />
<p class="modest" align="center">Figure 15: Packet details of the SUBSCRIBE request sent from UE1.</p>

Upon receiving the SUBSCRIBE request, the S-CSCF notifies the UE of its registration status through a NOTIFY request. Figure 16 shows the NOTIFY request in Step 6 of Figure 14. The `aor` attribute of the `registration` element loaded in the message body in XML format is set to the UE1’s public user identity, and the `state` attribute is set to `active`. In addition, the `uri` element in the `contact` element is set to the contact address bound to its public user identity, and the `unknown-param` element is set to `+g.3gpp.smsip`. Through this request, UE1 recognizes that its public user identity is active and that it can use the SMS over IMS service.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure16.png" />
<p class="modest" align="center">Figure 16: Details of the registration information on the NOTIFY request.</p>

### Transfer of a Short Message

Once the UE recognizes that the SMS over IMS service is available, it sends short messages using the SIP. Figure 17 shows the SIP packets when UE1 sends a short message to UE2, and Figure 18 shows the sequences. Furthermore, Figure 19 shows the MESSAGE request in Step 1 of Figure 18. In SMS over IMS, SMS PDUs are encapsulated in the SIP and transferred. Therefore, as shown in Figure 19, SMS-SUBMIT is loaded into the message body of the request. In addition, the Accept-Contact header is set to `+g.3gpp.smsip`, indicating the use of the SMS over IMS service. This MESSAGE request is forwarded to S-CSCF via P-CSCF.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure17.png" />
<p class="modest" align="center">Figure 17: Packets captured when UE1 sends the short message.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure18.png" />
<p class="modest" align="center">Figure 18: Sequence diagram of short message submission.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure19.png" />
<p class="modest" align="center">Figure 19: Packet details of the MESSAGE request submitted from UE1.</p>

Upon receiving the MESSAGE request, the S-CSCF forwards it to the SMSC with headers including the originating number. Because the Content-Type header of the received request (Figure 19) is set to `application/vnd.3gpp.sms`, the S-CSCF forwards it to the SMSC according to the iFC in Figure 11. Figure 20 shows the MESSAGE request in Step 3 of Figure 18. In the request, the P-Served-User header, From header, and P-Asserted-Identity header specify a public user identity that includes UE1’s phone number `818001234567`. It is inferred that the SMSC receiving this request determines the originating number of the short message based on the headers.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure20.png" />
<p class="modest" align="center">Figure 20: Packet details of the MESSAGE request submitted to the SMSC.</p>

The SMSC determines the originating number of the short message through a process and loads it into the SMS-DERIVER. Figure 21 shows the SIP packets when the SMSC delivered a short message from UE1 to UE2, and Figure 22 shows the sequences. In addition, Figure 23 shows the MESSAGE request in Step 1 of Figure 22. At request, the SMS-DERIVER is loaded into the message body. The TP-Originating-Address (TP-OA) field of the SMS-DERIVER is loaded with UE1’s phone number `818001234567`, the originating number. To clarify how the value of the TP-OA field is determined, the SMSC process is to be reviewed.

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure21.png" />
<p class="modest" align="center">Figure 21: Packets captured when the SMSC delivers the short message.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure22.png" />
<p class="modest" align="center">Figure 22: Sequence diagram of short message delivery.</p>

<img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure23.png" />
<p class="modest" align="center">Figure 23: Packet details of the MESSAGE request delivered from SMSC.</p>

### Decision Process at SMSC

The SMSC temporarily stores the submitted short message in its database and constructs the delivered message based on this data. Figure 24 shows the message temporarily stored in the SMSC database upon receiving SMS-SUBMIT from UE1. It can be seen that UE1’s phone number `818001234567`, the originator, is stored in the `caller` column and UE2’s phone number `819001234567`, the destination, is stored in the `callee`. The SMSC loads the value in the `caller` column into the TP-OA field in the SMS-DERIVER.

<p align="center"><img src="/assets/2023/decision_procedure_for_originating_numbers_in_sms_over_ims/28_figure24.png" width="600px" /></p>
<p class="modest" align="center">Figure 24: Short message stored in the SMSC database.</p>

Kamailio’s SMSC inserts the submitted short message into its database using native scripts in its configuration file. Figure 25 shows the insert script used in this test. The insert statement in the script inserts the value of the `$(avp(from){s.escape.common})` variable into the `caller` column<sup id="f14">[¹⁴](#fn14)</sup>. This variable is defined by the `$avp(from) = $(ai{uri.user});` script in the same file<sup id="f15">[¹⁵](#fn15)</sup>. In Kamailio’s native scripts, the `$ai` pseudo-variable allows retrieving a URI specified in a P-Asserted-Identity header of a SIP request<sup id="f16">[¹⁶](#fn16)</sup>. Furthermore, the `uri.user` function allows retrieving the userinfo part of a SIP URI<sup id="f17">[¹⁷](#fn17)</sup>. This means that if the value of the header is `sip:818001234567@ims.mnc001.mcc001.3gppnetwork.org`, `818001234567` will be extracted. Thus, the phone number contained in the public user identity specified in the P-Asserted-Identity header of the MESSAGE request is determined to be the originator. This decision process is consistent with the note in section 5.3.1.2 of 3GPP TS 24.341 mentioned earlier.

```mysql
sql_query("sms", "
    insert into messages (caller, callee, text, valid) 
    values (
        '$(avp(from){s.escape.common})', 
        '$(avp(to){s.escape.common})', 
        '$(avp(text){s.escape.common})', 
        now());")
```
<p class="modest" align="center">Figure 25: Script to insert short messages into the SMSC database.</p>

## Conclusion and Future Work

The originating numbers in SMS over IMS are determined based on the P-Asserted-Identity header of the SIP MESSAGE request encapsulating SMS-SUBMIT. By extracting the originating number from the public user identity specified in this header, SMSCs can load it into the TP-OA field of the SMS-DERIVER. However, the answer in this article is derived from testing Kamailio’s SMSC on a private mobile network. The 3GPP technical specifications do not strictly define the decision process for the originating numbers in SMSCs. Therefore, this ultimate decision is left to SMSC implementation. Future work is expected to test other SMSC software and mobile networks. Moreover, the decision procedure for originating numbers in Rich Communication Services (RCS), which also uses IMS, needs to be clarified.

---

<sup id="fn1">[¹](#f1)</sup> [3GPP TS 23.228,  IP Multimedia Subsystem (IMS); Stage 2](https://www.3gpp.org/DynaReport/23228.htm)  
<sup id="fn2">[²](#f2)</sup> [3GPP TS 24.341, Support of SMS over IP networks; Stage 3](https://www.3gpp.org/dynareport/24341.htm)  
<sup id="fn3">[³](#f3)</sup> [3GPP TS 23.204, Support of Short Message Service (SMS) over generic 3GPP Internet Protocol (IP) access; Stage 2](https://www.3gpp.org/DynaReport/23204.htm)  
<sup id="fn4">[⁴](#f4)</sup> [The 3G IP Multimedia Subsystem (IMS): Merging the Internet and the Cellular Worlds, 3rd Edition - Wiley](https://www.wiley.com/en-us/The+3G+IP+Multimedia+Subsystem+%28IMS%29%3A+Merging+the+Internet+and+the+Cellular+Worlds%2C+3rd+Edition-p-9781119964414)  
<sup id="fn5">[⁵](#f5)</sup> [RFC 3325 - Private Extensions to the Session Initiation Protocol (SIP) for Asserted Identity within Trusted Networks](https://datatracker.ietf.org/doc/html/rfc3325)  
<sup id="fn6">[⁶](#f6)</sup> [herlesupreeth/docker_open5gs - GitHub](https://github.com/herlesupreeth/docker_open5gs)  
<sup id="fn7">[⁷](#f7)</sup> [Open5GS - Open Source implementation for 5G Core and EPC](https://open5gs.org/)  
<sup id="fn8">[⁸](#f8)</sup> [srsRAN Project - Open Source RAN](https://www.srslte.com/)  
<sup id="fn9">[⁹](#f9)</sup> [Kamailio SIP Server](https://www.kamailio.org/w/)  
<sup id="fn10">[¹⁰](#f10)</sup> [SysmoISIM-SJA2 - Cellular Network Infrastructure - Osmocom](https://osmocom.org/projects/cellular-infrastructure/wiki/SysmoISIM-SJA2)  
<sup id="fn11">[¹¹](#f11)</sup> [pySim WiKi - Osmocom](https://osmocom.org/projects/pysim/wiki)  
<sup id="fn12">[¹²](#f12)</sup> [LimeSDR Mini v2.2 Board 23.01 documentation - MyriadRF](https://limesdr-mini.myriadrf.org/index.html)  
<sup id="fn13">[¹³](#f13)</sup> [herlesupreeth/CoIMS_Wiki - GitHub](https://github.com/herlesupreeth/CoIMS_Wiki)  
<sup id="fn14">[¹⁴](#f14)</sup> [docker_open5gs/kamailio_smsc.cfg#L338 - GitHub](https://github.com/herlesupreeth/docker_open5gs/blob/bd2501bc11ee344588b1079f21b119f4d3656a7d/smsc/kamailio_smsc.cfg#L338)  
<sup id="fn15">[¹⁵](#f15)</sup> [docker_open5gs/kamailio_smsc.cfg#L166 - GitHub](https://github.com/herlesupreeth/docker_open5gs/blob/bd2501bc11ee344588b1079f21b119f4d3656a7d/smsc/kamailio_smsc.cfg#L166)  
<sup id="fn16">[¹⁶](#f16)</sup> [Kamailio SIP Server v5.3.x (stable): Pseudo-Variables](https://www.kamailio.org/wikidocs/cookbooks/5.3.x/pseudovariables/#ai-uri-inp-asserted-identity-header)  
<sup id="fn17">[¹⁷](#f17)</sup> [Kamailio SIP Server v5.3.x (stable): Transformations](https://www.kamailio.org/wikidocs/cookbooks/5.3.x/transformations/#uriuser)
