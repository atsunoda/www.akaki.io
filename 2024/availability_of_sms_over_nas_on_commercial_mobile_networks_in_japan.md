---
description: A previous article demonstrated short message service (SMS) over the Non-Access Stratum (NAS) in a private mobile network. However, as noted in another previous article, SMS can also be transported via the IP Multimedia Subsystem (IMS) using the Session Initiation Protocol (SIP). Under what circumstances is SMS over NAS used on commercial mobile networks provided by mobile network operators (MNOs)? This article examines the availability of SMS over NAS by capturing packets of short messages sent on commercial mobile networks in Japan.
---

# Availability of SMS over NAS on Commercial Mobile Networks in Japan

<p class="modest" align="left">Dec 3, 2024</p>

---

A previous [article](/2024/decision_procedure_for_originating_numbers_in_sms_over_nas.md) demonstrated short message service (SMS) over the Non-Access Stratum (NAS) in a private mobile network. However, as noted in another previous [article](/2023/decision_procedure_for_originating_numbers_in_sms_over_ims.md), SMS can also be transported via the IP Multimedia Subsystem (IMS) using the Session Initiation Protocol (SIP). Under what circumstances is SMS over NAS used on commercial mobile networks provided by mobile network operators (MNOs)? This article examines the availability of SMS over NAS by capturing packets of short messages sent on commercial mobile networks in Japan.

## Summary

The availability of SMS over NAS in Japan depends on the MNO, as shown in Table 1. Since IMS is enabled on Japanese MNOs’ mobile networks, when sending short messages from the default Messages app on Android, three of the four operators used SMS over IMS, while one used SMS over NAS. In contrast, when sending from the modem of the Android device using AT commands, even operators that used SMS over IMS also used SMS over NAS. The availability of SMS over NAS depends on the MNO’s network configuration and was confirmed to be available even when IMS is enabled.

<p class="modest" align="center">Table 1: SMS transport methods used by Japanese MNOs.</p>

|                  | NTT Docomo   | KDDI         | SoftBank     | Rakuten      |
| :--------------: | :----------: | :----------: | :----------: | :----------: |
| **Messages app** | SMS over IMS | SMS over IMS | SMS over NAS | SMS over IMS |
| **AT commands**  | SMS over NAS | N/A          | SMS over NAS | SMS over NAS |

## Capturing SMS Packets

To identify the SMS transport method, tests were conducted on the networks of four Japanese MNOs: NTT Docomo, KDDI, SoftBank, and Rakuten. These networks are 5G Non-Standalone (NSA) and IMS enabled. The SIM cards were inserted into a non-rooted Galaxy S24, and packets were captured while sending short messages. Messages were sent using the default Messages app on Android, and AT commands; Rich Communication Services (RCS) was disabled in the Messages app.

The SMS packets were captured using SCAT, a tool that captures packets carried by radio signals based on diagnostic messages from Qualcomm and Samsung modems<sup id="f1">[¹](#fn1)</sup>. The Galaxy S24 (SM-S921Q) used in this test has a Qualcomm Snapdragon modem. Therefore, access to diagnostic messages was enabled by entering `*#0808#` in the Phone app and changing the USB setting to `RNDIS + DM + MODEM + ADB`. The Galaxy S24 was connected to a MacBook via USB, and SCAT was run to capture SMS packets, as shown in Figure 1.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure1.webp" width="770" height="158" decoding="async" alt="" />
<p class="modest" align="center">Figure 1: Running SCAT to capture SMS packets.</p>

## Sending SMS from the Messages App

When sending short messages from the Messages app, SMS Protocol Data Units (PDUs) were encapsulated into SIP packets and transported on the NTT Docomo and KDDI networks. Figure 2 shows a SIP packet captured on the NTT Docomo network; the SMS-SUBMIT is loaded in the message body of the SIP packet. To detect the SIP packet in Wireshark, the User Datagram Protocol (UDP) must be decoded as Internet Protocol version 6 (IPv6)<sup id="f2">[²](#fn2)</sup>. This confirms that SMS over IMS is used.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure2.webp" width="770" height="563" decoding="async" alt="" />
<p class="modest" align="center">Figure 2: SMS PDU encapsulated in the SIP packet.</p>

The Rakuten network also transported short messages using SMS over IMS. However, no SIP packets containing SMS PDUs were observed; instead, as shown in Figure 3, Encapsulating Security Payload (ESP) packets were captured when a short message was sent. This suggests that the SMS PDU is encrypted in one of these packets. Since decrypting ESP packets is beyond the scope of this test, the SMS transport method was identified using Android logs. Figure 4 shows the output from the `logcat` command when a short message was sent. These logs confirm that SMS over IMS is used.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure3.webp" width="770" height="235" decoding="async" alt="" />
<p class="modest" align="center">Figure 3: ESP packets captured when a short message was sent.</p>

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure4.webp" width="770" height="188" decoding="async" alt="" />
<p class="modest" align="center">Figure 4: Logs output when a short message was sent.</p>

In contrast, the SoftBank network transported short messages using SMS over NAS. Figure 5 shows a NAS packet captured when a short message was sent; the SMS-SUBMIT is loaded in the message container of the NAS packet. As noted in the previous [article](/2023/decision_procedure_for_originating_numbers_in_sms_over_ims.md), the `+g.3gpp.smsip` parameter must be set in the SIP REGISTER request to use SMS over IMS. While the other three operators had this parameter set, SoftBank did not. Therefore, it can be concluded that SMS over NAS is used even when IMS is enabled.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure5.webp" width="770" height="575" decoding="async" alt="" />
<p class="modest" align="center">Figure 5: SMS PDU encapsulated in the NAS packet.</p>

Operators other than SoftBank may also use SMS over NAS if they are not registered for IMS. Disabling IMS on an Android device results in deregistration. However, the Galaxy S24 lacks an option to manually disable IMS. As a workaround, the behavior of sending short messages were tested under the following condition: immediately after turning on airplane mode to deregister IMS and turning it off again, i.e., before IMS re-registration was completed. This test confirmed that SMS over NAS was used on the NTT Docomo network, whereas no evidence of its use was found on KDDI and Rakuten. Further analysis is required to clarify the factors influencing the Messages app’s selection of SMS transport methods.

## Sending SMS from AT Commands

When sending short messages from the modem embedded in the Android device using AT commands, the messages were transported using SMS over NAS. AT commands are used to control modems and other devices through a serial interface. As noted in a previous [article](/2022/transmission_and_detection_of_silent_sms_in_android.md), short messages can be sent using AT commands. In this test, SMS over NAS was confirmed on the NTT Docomo, SoftBank, and Rakuten networks. Figure 6 shows a NAS packet captured on the Rakuten network. Since NAS packets are not encrypted by ESP, it can be confirmed using Wireshark that the packet contains the SMS PDU.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure6.webp" width="770" height="648" decoding="async" alt="" />
<p class="modest" align="center">Figure 6: SMS PDU sent using AT commands.</p>

However, when attempting to send short messages using AT commands on the KDDI network, a `+CMS ERROR: 500` was returned. Since the packets captured at this time do not contain an SMS PDU, it is clear that no message was sent from the modem. According to 3GPP TS 24.301, which defines the NAS specification for the Evolved Packet System (EPS), the use of SMS in the EPS requires the user equipment to set `SMS only` in the `Additional update type` element of the Attach request, and the core network to set `SMS only` in the `Additional update result` element of the Attach accept<sup id="f3">[³](#fn3)</sup>. As shown in Figure 7, `SMS only` was set in the Attach request when connecting to all MNO networks. Furthermore, `SMS only` was also set in the Attach accept from the NTT Docomo, SoftBank, and Rakuten networks, as shown in Figure 8. In contrast, the Attach accept from the KDDI network did not have the `Additional update result` element as shown in Figure 9. Therefore, it can be concluded that the error occurred because KDDI’s EPS does not provide SMS. Based on these results, the use of SMS over NAS on the KDDI network could not be confirmed.

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure7.webp" width="770" height="504" decoding="async" alt="" />
<p class="modest" align="center">Figure 7: Attach request to the NTT Docomo network.</p>

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure8.webp" width="770" height="431" decoding="async" alt="" />
<p class="modest" align="center">Figure 8: Attach accept from the NTT Docomo network.</p>

<img src="/assets/2024/availability_of_sms_over_nas_on_commercial_mobile_networks_in_japan/30_figure9.webp" width="770" height="358" decoding="async" alt="" />
<p class="modest" align="center">Figure 9: Attach accept from the KDDI network.</p>

## Conclusion and Future Work

Test results show that the availability of SMS over NAS depends on the MNO’s network configuration. When IMS is enabled, short messages sent from the Messages app on Android were transported using SMS over IMS, except for one MNO’s network, which used SMS over NAS. In contrast, messages sent from the modem using AT commands were consistently transported using SMS over NAS. However, these results are based on testing conducted on 5G NSA networks in Japan. SMS over NAS should be further tested on 5G Standalone (SA) networks, as it may also be supported on 5G Core (5GC). Additionally, testing with iOS devices is expected, as different mobile operating systems may handle SMS transport differently.

---

<sup id="fn1">[¹](#f1)</sup> [fgsect/scat: SCAT: Signaling Collection and Analysis Tool - GitHub](https://github.com/fgsect/scat)  
<sup id="fn2">[²](#f2)</sup> [11.4.2. User Specified Decodes - Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/ChCustProtocolDissectionSection.html#ChAdvDecodeAs)  
<sup id="fn3">[³](#f3)</sup> [3GPP TS 24.301, Non-Access-Stratum (NAS) protocol for Evolved Packet System (EPS); Stage 3](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=1072)
