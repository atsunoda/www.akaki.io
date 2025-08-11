# Reproducing SMS over NAS Spoofing in a Private 5G Mobile Network

<time datetime="2025-02-27">Feb 27, 2025</time>

---

I recently wrote articles [1](/2024/decision_procedure_for_originating_numbers_in_sms_over_nas.md), [2](/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan.md), and [3](/2025/sending_arbitrary_sms_over_nas_messages_with_an_extended_srsue.md), which focused on short message service (SMS) over Non-Access Stratum (NAS). A previous study has shown that SMS over NAS can be exploited for origin spoofing in Long Term Evolution (LTE) networks. Does the same spoofability exist in 5G mobile networks? In this article, I experimentally reproduce SMS over NAS spoofing in a private 5G mobile network based on the methods presented in the previous study.

## Summary

I have confirmed that the spoofability of SMS over NAS also exists in 5G mobile networks. In the previous study, an attacker spoofed a Radio Resource Control (RRC) connection to the evolved Node B (eNB) and sent a fake SMS over NAS message to the Mobility Management Entity (MME). To reproduce this approach in a private 5G mobile network, I spoofed an RRC connection to the next generation Node B (gNB) and sent a fake SMS over NAS message to the Access and Mobility Management Function (AMF). As a result, when the NAS security protection by the AMF was weak, an attacker was able to spoof the origin of an SMS message. Therefore, in 5G mobile networks where AMF validation is not properly performed, SMS over NAS could be exploited for origin spoofing.

## Approach Presented in the Previous Study

The previous study investigating the security of the LTE control plane has revealed the spoofability of SMS over NAS. Kim et al. developed a tool for dynamic security testing of LTE networks and demonstrated several attacks, including SMS spoofing, in commercial mobile networks<sup id="f1">[¹](#fn1)</sup>. The SMS spoofing approach presented in their study consists of two key phases: RRC connection spoofing and NAS message sending. This approach requires the attacker to know the S-Temporary Mobile Subscriber Identity (S-TMSI) assigned to the user equipment (UE) of victim 1 (the legitimate sender), and the phone number of victim 2 (the receiver). Therefore, the flow of the SMS spoofing approach demonstrated in the previous study can be illustrated as shown in Figure 1.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure1.webp" width="400" height="320" decoding="async" alt="" /><figcaption>Figure 1: Flowchart of the SMS spoofing approach.</figcaption></figure>

In the first phase, the attacker sets the S-TMSI of victim 1 in the RRC Connection Request sent from their own UE, impersonating victim 1 to establish a connection with the LTE network. According to 3GPP TS 36.331<sup id="f2">[²](#fn2)</sup>, which defines the LTE RRC specifications, the UE and eNB establish the RRC connection before the Evolved Packet Core (EPC) provides any UE context information. Consequently, an attacker can establish an RRC connection with an eNB of a specific mobile network operator (MNO) without being a subscriber of that MNO. Furthermore, in the LTE RRC connection establishment procedure, if the upper layer (in this context, the NAS layer) provides an S-TMSI, the RRC Connection Request is required to specify the S-TMSI in the `ue-Identity` element. Due to this design, if an attacker can obtain the S-TMSI of a legitimate subscriber, they can impersonate that subscriber to establish an RRC connection.

In the next phase, after successfully establishing an RRC connection using victim 1’s S-TMSI, the attacker sends a NAS message containing an SMS-SUBMIT directed to victim 2 through the LTE network. According to 3GPP TS 24.301<sup id="f3">[³](#fn3)</sup>, which defines the LTE NAS specifications, integrity protection and encryption are defined for NAS security. Encryption is optional, while integrity protection becomes mandatory once the security context is established through the NAS registration procedure. Therefore, before initiating the NAS registration procedure, specifically immediately after successfully establishing an RRC connection using victim 1’s S-TMSI, the attacker may successfully deliver an SMS message that appears to originate from victim 1’s phone number by sending an Uplink NAS transport containing an SMS-SUBMIT to the AMF.

In the previous study, the spoofability of SMS over NAS was tested by sending Uplink NAS transports using three different message types. The first method was a plain NAS message without security protection, the second was an integrity-protected NAS message with an invalid message authentication code (MAC), and the third was a replayed NAS message. By using these NAS messages to send SMS-SUBMIT, SMS spoofing was demonstrated in commercial LTE networks. In the following chapters, I reproduce SMS spoofing using an integrity-protected NAS message with an invalid MAC in a private 5G mobile network. Since the tool developed in the previous study has not been publicly released, another tool was used to reproduce the spoofing approach.

## Reproduction Environment and Tools

The spoofability of SMS over NAS in 5G was reproduced in a mobile network built with srsRAN<sup id="f4">[⁴](#fn4)</sup> and Open5GS<sup id="f5">[⁵](#fn5)</sup>. The reproduction environment reused the mobile network described in the previous [article](/2024/decision_procedure_for_originating_numbers_in_sms_over_nas.md) along with the subscriber information presented in a different previous [article](/2023/decision_procedure_for_originating_numbers_in_sms_over_ims.md). A Pixel 6 was used as the legitimate sender (victim 1, UE1) of SMS messages, while a Galaxy S24 was used as the receiver (victim 2, UE2). As the spoofed sender (attacker, UE3), the srsUE extension “smsUE<sup id="f6">[⁶](#fn6)</sup>” described in the previous [article](/2025/sending_arbitrary_sms_over_nas_messages_with_an_extended_srsue.md) was used. The deployed mobile network used as the reproduction environment is shown in Figure 2. UE1 was assigned the International Mobile Subscriber Identity (IMSI) `001010000000001` and the phone number `818001234567`, while UE2 was assigned `001010000000002` and `819001234567`. UE3, impersonating UE1, was not registered as a subscriber.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure2.webp" width="500" height="310" decoding="async" alt="" /><figcaption>Figure 2: Mobile network deployed as the reproduction environment.</figcaption></figure>

By modifying its configuration, smsUE can be adapted to reproduce the SMS spoofing approach. smsUE is designed to send the SMS-SUBMIT specified in the `sms.pdu` option, and by default, it sends SMS messages after completing the NAS registration procedure. However, if the 5G-S-TMSI used for RRC connection establishment is specified in the `rrc.nr_ue_id` option, smsUE sends the SMS message before initiating the NAS registration procedure. Since smsUE enables integrity protection by default, when an SMS message is sent before initiating the NAS registration procedure, an invalid MAC is applied. These specifications enable smsUE to reproduce the spoofing approach demonstrated in the previous study. In the following chapters, I report on the results of reproducing the approach using smsUE.

## RRC Connection Spoofing

To reproduce RRC connection spoofing, the 5G-S-TMSI was specified during the RRC connection establishment procedure in 5G. Figure 3 shows the sequence of the RRC connection establishment procedure based on 3GPP TS 38.331<sup id="f7">[⁷](#fn7)</sup>, which defines the RRC specifications for 5G NR, and 3GPP TS 38.413<sup id="f8">[⁸](#fn8)</sup>, which defines the NGAP specifications. According to 3GPP TS 38.331, two elements are defined for the UE to present the 5G-S-TMSI to the gNB: the `ue-Identity` element in the RRC Setup Request and the `ng-5G-S-TMSI-Value` element in the RRC Setup Complete. When the gNB receives the 5G-S-TMSI from the UE, it includes the provided value in an Initial UE Message and forwards it to the AMF, following the specifications of 3GPP TS 38.413.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure3.webp" width="500" height="250" decoding="async" alt="" /><figcaption>Figure 3: Sequence of RRC connection establishment in 5G.</figcaption></figure>

The spoofed sender, UE3, establishes an RRC connection by specifying the 5G-S-TMSI assigned to the legitimate sender, UE1. For example, if UE1’s 5G-S-TMSI is `0x40c000063b`, smsUE, acting as UE3, can impersonate UE1 and establish an RRC connection by using the `--rrc.nr_ue_id 40c000063b` command-line option. The packets captured when UE3 spoofed the RRC connection are shown in Figures 4–6, and the Open5GS logs are shown in Figure 7. In Figure 4, part of UE1’s 5G-S-TMSI is set in the `ue-Identity` element of the RRC Setup Request. Specifically, ng-5G-S-TMSI-Part1 corresponds to the rightmost 39 bits of the 5G-S-TMSI. As a result, the configured value appears different in hexadecimal notation but matches in binary notation. Furthermore, Figure 5 shows that the `ng-5G-S-TMSI-Value` element in the RRC Setup Complete contains UE1’s 5G-S-TMSI, while Figure 6 shows that the `FiveG-S-TMSI` element in the Initial UE Message also contains UE1’s 5G-S-TMSI. As shown in the logs in Figure 7, the Initial UE Message was recognized by the AMF as a connection from UE1. Therefore, UE3 successfully established an RRC connection by impersonating UE1.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure4.webp" width="770" height="202" decoding="async" alt="" /><figcaption>Figure 4: Packet details of the RRC Setup Request.</figcaption></figure>

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure5.webp" width="770" height="244" decoding="async" alt="" /><figcaption>Figure 5: Packet details of the RRC Setup Complete.</figcaption></figure>

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure6.webp" width="770" height="366" decoding="async" alt="" /><figcaption>Figure 6: Packet details of the Initial UE Message.</figcaption></figure>

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure7.webp" width="770" height="56" decoding="async" alt="" /><figcaption>Figure 7: AMF logs when the Initial UE Message is received.</figcaption></figure>

## NAS Message Sending

Immediately after spoofing the RRC connection, a NAS message containing an SMS-SUBMIT was sent to the AMF. The SMS-SUBMIT included in the Uplink NAS transport was specified in the smsUE configuration file, as shown in Figure 8. The TP-Destination-Address field of the SMS-SUBMIT was set to UE2’s phone number `819001234567`, and the TP-User-Data field contained the message text `fake hello`. The spoofability of SMS over NAS was tested by sending this Uplink NAS transport with an invalid MAC.

```
[sms]
pdu = 5901200084000581005155f516013d0c8118091032547600000ae6f0ba0c4297d9ec37
```
<figure><figcaption>Figure 8: sms.pdu option in smsUE configuration file.</figcaption></figure>

When the 5G-S-TMSI for RRC connection establishment is specified, smsUE protects the Uplink NAS transport with an invalid MAC. According to 3GPP TS 33.501<sup id="f9">[⁹](#fn9)</sup>, which defines the security architecture of 5GS, the integrity of NAS messages is verified by checking whether the MAC calculated by the UE matches the one calculated by the AMF. However, when the `rrc.nr_ue_id` option is set, smsUE sends the Uplink NAS transport before initiating the NAS registration procedure, making it impossible to generate a valid MAC that matches the one calculated by the AMF. Therefore, as shown in Figure 9, smsUE sets the MAC to `0x00000000` when sending the Uplink NAS transport. Figure 9 also shows that the payload of the Uplink NAS transport contains the SMS Protocol Data Unit (PDU) shown in Figure 8.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure9.webp" width="770" height="549" decoding="async" alt="" /><figcaption>Figure 9: Packet details of the Uplink NAS transport.</figcaption></figure>

However, the Uplink NAS transport with an invalid MAC was rejected by the AMF due to MAC verification failure. As shown in the logs in Figure 10, the AMF rejected the received Uplink NAS transport with the error message `ERROR: No Security Context`. Figure 11 shows the case handling in [`/src/amf/gmm-sm.c`](https://github.com/open5gs/open5gs/blob/1b21eba81ed7740317ec820ce96dda3b587ef4e9/src/amf/gmm-sm.c#L1650-L1655), where this error occurred. In this case, validation failed at the `SECURITY_CONTEXT_IS_VALID()` function. This function, defined in [`/src/amf/context.h`](https://github.com/open5gs/open5gs/blob/1b21eba81ed7740317ec820ce96dda3b587ef4e9/src/amf/context.h#L356-L360), checks the `mac_failed` flag, as shown in Figure 12. As shown in the AMF log in Figure 10, `WARNING: NAS MAC verification failed(0x0 != 0xc64d6496)`, the MAC verification failed, resulting in the rejection of the Uplink NAS transport with an invalid MAC. Therefore, in a 5G mobile network using Open5GS, SMS spoofing using NAS messages with an invalid MAC could not be reproduced.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure10.webp" width="770" height="56" decoding="async" alt="" /><figcaption>Figure 10: AMF logs when the Uplink NAS transport is received.</figcaption></figure>

```c
       case OGS_NAS_5GS_UL_NAS_TRANSPORT:
            if (!h.integrity_protected || !SECURITY_CONTEXT_IS_VALID(amf_ue)) {
                ogs_error("No Security Context");
                OGS_FSM_TRAN(s, gmm_state_exception);
                break;
            }
```
<figure><figcaption>Figure 11: Case handling in /src/amf/gmm-sm.c where the error occurred.</figcaption></figure>

```
#define SECURITY_CONTEXT_IS_VALID(__aMF) \
    ((__aMF) && \
    ((__aMF)->security_context_available == 1) && \
     ((__aMF)->mac_failed == 0) && \
     ((__aMF)->nas.ue.ksi != OGS_NAS_KSI_NO_KEY_IS_AVAILABLE))
```
<figure><figcaption>Figure 12: SECURITY_CONTEXT_IS_VALID() function defined in /src/amf/context.h.</figcaption></figure>

## Demonstration of SMS over NAS Spoofing

To reproduce SMS over NAS spoofing, MAC verification was removed from the AMF. In Open5GS, the AMF checks the validity of the MAC in received NAS messages using the `nas_5gs_security_decode()` function in [`/src/amf/nas-security.c`](https://github.com/open5gs/open5gs/blob/6a2225bb680cd36cee8ea65ee8d4483c7988982a/src/amf/nas-security.c#L169-L173). To prevent the AMF from verifying the MAC of received NAS messages, the MAC verification code was removed, as shown in Figure 13. As a result of this code modification, the `mac_failed` flag shown in Figure 12 is no longer set, allowing NAS messages with an invalid MAC to be accepted by the AMF. This condition reproduces the vulnerable MME demonstrated in the previous study when SMS spoofing was successful.

```diff
        if (security_header_type.integrity_protected) {
            uint8_t mac[NAS_SECURITY_MAC_SIZE];
            uint32_t mac32;
            uint32_t original_mac = h->message_authentication_code;

            /* calculate NAS MAC(message authentication code) */
            ogs_nas_mac_calculate(amf_ue->selected_int_algorithm,
                amf_ue->knas_int, amf_ue->ul_count.i32,
                amf_ue->nas.access_type,
                OGS_NAS_SECURITY_UPLINK_DIRECTION, pkbuf, mac);
            h->message_authentication_code = original_mac;

            memcpy(&mac32, mac, NAS_SECURITY_MAC_SIZE);
-           if (h->message_authentication_code != mac32) {
-               ogs_warn("NAS MAC verification failed(0x%x != 0x%x)",
-                   be32toh(h->message_authentication_code), be32toh(mac32));
-               amf_ue->mac_failed = 1;
-           }
        }
```
<figure><figcaption>Figure 13: MAC verification removed from nas_5gs_security_decode() function in /src/amf/nas-security.c.</figcaption></figure>

By reproducing a vulnerable environment where AMF does not verify the MAC, SMS over NAS spoofing was successfully demonstrated. Figure 14 shows a demonstration in which UE3 (smsUE) impersonates UE1 (Pixel 6) and sends an SMS message to UE2 (Galaxy S24). In the figure, UE1 first sends an SMS message with the text `hello` to UE2. Then, UE3 uses UE1’s 5G-S-TMSI (part of the 5G-GUTI) to send an SMS message with the text `fake hello`. As a result, both SMS messages appear in the same conversation thread on UE2. This means that the originating number is recognized as UE1’s phone number `818001234567`. Therefore, UE3 successfully impersonated UE1 and sent the fake SMS message.

<figure><video controls muted playsinline poster="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure14.webp" src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure14.mp4" type="video/mp4"></video><figcaption>Figure 14: Demonstration of SMS over NAS spoofing.</figcaption></figure>

The spoofability of SMS over NAS was demonstrated by comparing the origin of the delivered SMS PDUs. Figure 15 shows the packet of the legitimate SMS message received by UE2 from UE1, while Figure 16 shows the fake SMS message received from UE3. In both cases, the TP-Originating-Address field in the SMS-DELIVER is set to UE1’s phone number `818001234567`. This result is consistent with my previous study, in which I demonstrated the spoofability of SMS using SMPP<sup id="f10">[¹⁰](#fn10)</sup>. Therefore, it is concluded that the spoofability of SMS over NAS also exists in 5G mobile networks.

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure15.webp" width="770" height="284" decoding="async" alt="" /><figcaption>Figure 15: Packet details of the SMS-DELIVER from UE1.</figcaption></figure>

<figure><img src="/assets/2025/reproducing_sms_over_nas_spoofing_in_a_private_5g_mobile_network/32_figure16.webp" width="770" height="284" decoding="async" alt="" /><figcaption>Figure 16: Packet details of the SMS-DELIVER from UE3.</figcaption></figure>

## Conclusion and Future Work

SMS over NAS spoofing, demonstrated in the previous study in 4G mobile networks, was successfully reproduced in a 5G mobile network. To reproduce the behavior of the vulnerable MME identified in the previous study, MAC verification had to be removed from the AMF in a private 5G mobile network. Further research is needed to determine whether such vulnerable AMFs are deployed in commercial mobile networks. In addition, methods for stealing the legitimate sender’s 5G-S-TMSI need to be investigated. The previous study presented paging sniffing and the use of malicious base stations (also known as IMSI catchers) as methods for capturing a victim’s S-TMSI. Evaluating the effectiveness of these methods in 5G mobile networks, as well as exploring alternative approaches for stealing 5G-S-TMSI, remains a topic for future research.

---

<sup id="fn1">[¹](#f1)</sup> [Hongil Kim et al. 2019. Touching the Untouchables: Dynamic Security Analysis of the LTE Control Plane](https://doi.org/10.1109/SP.2019.00038)  
<sup id="fn2">[²](#f2)</sup> [3GPP TS 36.331, Evolved Universal Terrestrial Radio Access (E-UTRA); Radio Resource Control (RRC); Protocol specification](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=2440)  
<sup id="fn3">[³](#f3)</sup> [3GPP TS 24.301, Non-Access-Stratum (NAS) protocol for Evolved Packet System (EPS); Stage 3](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=1072)  
<sup id="fn4">[⁴](#f4)</sup> [srsRAN Project - Open Source RAN](https://www.srsran.com/)  
<sup id="fn5">[⁵](#f5)</sup> [Open5GS - Open Source implementation for 5G Core and EPC](https://open5gs.org/)  
<sup id="fn6">[⁶](#f6)</sup> [atsunoda/smsUE: Extended srsUE that supports 5G SMS over NAS - GitHub](https://github.com/atsunoda/smsUE/)  
<sup id="fn7">[⁷](#f7)</sup> [3GPP TS 38.331, NR; Radio Resource Control (RRC); Protocol specification](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3197)  
<sup id="fn8">[⁸](#f8)</sup> [3GPP TS 38.413, NG-RAN; NG Application Protocol (NGAP)](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3223)  
<sup id="fn9">[⁹](#f9)</sup> [3GPP TS 33.501, Security architecture and procedures for 5G System](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3169)  
<sup id="fn10">[¹⁰](#f10)</sup> [Akaki Tsunoda. 2024. Demonstrating Spoofability of an Originating Number when Sending an SMS using SMPP](https://doi.org/10.1145/3615667)
