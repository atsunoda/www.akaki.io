# Analysis and Reproduction of Spoofed SMS-DELIVER

<p class="modest" align="left">Mar 21, 2022</p>

---

As past [articles 1](/2019/sms_spoofing.md) (in Japanese) and [2](/2019/sms_spoofing_2.md) (in Japanese) have indicated, the originating display name (called the “sender ID”) of the SMS can be spoofed. However, when I reviewed the SMS technical specifications in a previous [article](/2022/transmission_and_detection_of_silent_sms_in_android.md), SMS-SUBMIT did not have a field to load the sender ID. How are the alphanumeric characters of the sender ID transferred? This study reveals the answer to this question by analyzing the spoofed SMS-DELIVER.

## Summary

The sender ID is submitted to the SMSC by the protocol for the transfer of short messages and is loaded into the TP-OA (TP-Originating-Address) of the SMS-DELIVER. By analyzing the spoofed SMS-DELIVER, the alphanumeric characters of the sender ID are found to be included in the TP-OA. Because the TP-OA cannot be loaded in the SMS-SUBMIT, the sender ID cannot be specified in the SMS protocol defined by the 3GPP. However, protocols for the transfer of short messages can specify an alphanumeric name as the originating address. Therefore, this article reproduced a message specifying the sender ID using SMPP (Short Message Peer-to-Peer).

## What is SMS Spoofing

Before proceeding, let us remind ourselves of SMS spoofing. SMS spoofing is the act of deceiving recipients by spoofing the sender of a short message, and is a technique used in phishing attacks via SMS. Generally, SMS clients display messages from the same sender on the same thread. This specification allows an attacker to insert a phishing message spoofed as a legitimate brand sender ID in a legitimate thread. For example, Figure 1 shows a phishing message spoofed as Amazon, as observed in Japan. The message in the green box is real and that in the red box is fake<sup id="f1">[¹](#fn1)</sup>; thus, SMS spoofing can be a factor in the success of phishing attacks.

<p align="center"><img src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure1.png" width="300px" /></p>
<p class="modest" align="center">Figure 1: Phishing message spoofed as Amazon.</p>

## Analysis of SMS-DELIVER

To analyze spoofed SMS-DELIVER, a short message spoofing the sender ID must be sent as a sample. As indicated in the aforementioned articles (in Japanese), some SMS gateways allow any alphanumeric character to be specified for the sender ID. This study uses Twilio to send messages spoofing the sender ID as `Amazon` and analyzes the received SMS-DELIVER. The gateway provider’s terms of service and local laws may prohibit the act of deceiving others through SMS. Therefore, the test only sends messages to my own phone number.

The spoofed SMS-DELIVER is captured using an Android app that displays the received SMS PDUs. The source code of the Android app used in the test is shown in Figure 2. This app displays the results of executing `SmsMessage.getOriginatingAddress()`, `SmsMessage.getDisplayOriginatingAddress()`, and `SmsMessage.getPdu()` on received short messages. By granting `RECEIVE_SMS` permission to this app, I observe the SMS PDUs received by the Android device. Figure 3 illustrates what happens when the app receives a short message sent from Twilio. The SMS-DELIVER of a message can be captured with the sender ID specified as `Amazon`.

```kotlin
private fun receiveSms() {
    registerReceiver(object: BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            val sb = StringBuilder()
            for (sms in Telephony.Sms.Intents.getMessagesFromIntent(intent)) {
                sb.append("OriginatingAddress: " + sms.originatingAddress + "\n")
                sb.append("DisplayOriginatingAddress: " + sms.displayOriginatingAddress + "\n")
                sb.append("Pdu: ")
                for (byte in sms.pdu.toUByteArray()) {
                    sb.append(byte.toString(16).uppercase().padStart(2, '0'))
                }
            }
            findViewById<TextView>(R.id.textView).text = sb.toString()
        }
    }, IntentFilter("android.provider.Telephony.SMS_RECEIVED"))
}
```
<p class="modest" align="center">Figure 2: The source code of the Android app.</p>

<video controls src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure3.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 3: Capturing a spoofed SMS-DELIVER.</p>

From the captured SMS-DELIVER, it can be observed that the alphanumeric characters specified as the sender ID are loaded into the TP-OA. Figure 4 shows the data from the received SMS PDU minus the SCA (Service Center Address), that is, SMS-DELIVER. According to the SMS specification 3GPP TS 23.040<sup id="f2">[²](#fn2)</sup>, the Type-of-Address field in the TP-OA comprises a 1bit fixed value, 3bits Type-of-number, and 4bits Numbering-plan-identification. Section 9.1.2.5 of the specification assigns `101` as the bit pattern to indicate an alphanumeric representation in the Type-of-number. In addition, the alphanumeric characters of the sender ID are encoded in GSM 7-bit characters and included in the Address-Value field. Another question is how these alphanumeric characters are submitted to the SMSC.

<img src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure4.png" />
<p class="modest" align="center">Figure 4: Spoofed SMS-DELIVER.</p>

## Reproduction by SMPP

The SMS-SUBMIT specification defined by 3GPP TS 23.040 does not provide a way to declare TP-OA; however, there are other ways to submit short messages to the SMSC apart from SMS-SUBMIT. For example, there are methods to use SMPP, UCP/EMI, or CIMD, which are protocols for transferring short messages. Among them, SMPP is widely used, even in Twilio<sup id="f3">[³](#fn3)</sup>. The SMPP specification defines “submit_sm” as the PDU by which a message is submitted to the SMSC<sup id="f4">[⁴](#fn4)</sup>. Figure 5 illustrates the process of short message transfer via an SMPP gateway on the Internet.

<img src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure5.png" />
<p class="modest" align="center">Figure 5: Short message transfer by SMPP.</p>

Using the SMPP, short messages can be submitted with the sender ID to the SMSC. According to the SMPP specification, submit_sm specifies the source address. Figure 6 shows an example of submit_sm with the sender ID spoofed as `Amazon`. To declare the sender ID, specify `05` (bit pattern: `00000101`) in the source_addr_ton field. In addition, specify the alphanumeric characters in the source_addr field in hexadecimal and include the null terminator `00` at the end. The details of submit_sm are described in Section 4.4.1 of the specification.

<img src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure6.png" />
<p class="modest" align="center">Figure 6: Spoofed submit_sm.</p>

By sending SMPP PDUs, I attempt to reproduce short messages displaying a spoofed sender ID. Unfortunately, Twilio does not provide an SMPP gateway for users; hence, this study uses SMSGlobal’s gateway<sup id="f5">[⁵](#fn5)</sup>. The SMPP requires a session to be established between the client and server before sending messages; therefore, after establishing the session using a bind_transceiver PDU, the submit_sm PDUs are sent. This test connects to the gateway via Socat’s named pipe, and sends these PDUs to the same TCP stream. Figure 7 illustrates what happens when the PDUs are sent. After sending a spoofed submit_sm, the response indicated `REJECTED` and the Android device did not receive the short message. In contrast, when resending submit_sm with the source_addr field changed to `Amazoo`, the device received the message with the specified sender ID. This behavior indicates that SMSGlobal prohibits the specification of major brands as sender ID.

<video controls src="/assets/2022/analysis_and_reproduction_of_spoofed_sms-deliver/24_figure7.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 7: Sending a spoofed submit_sm.</p>

## Conclusion and Future Work

Because the SMPP allows short messages with a specified sender address to be submitted to the SMSC, it can be used to load an alphanumeric sender ID in the SMS-DELIVER. Consequently, if attackers can spoof the sender ID to the legitimate brand name, they can insert a fake message in the same thread as legitimate messages. This study focused on sender ID spoofing; therefore, future work should also research the possibility of originating phone number spoofing. This requires an understanding of how routing works, based on phone numbers on mobile networks. As other research focus, RCS (Rich Communication Services), which is expected to be the next generation of SMS, has the ability to verify the sender of a message<sup id="f6">[⁶](#fn6)</sup>. The reliability of this ability will also be a subject of future research.

---

<sup id="fn1">[¹](#f1)</sup> [Search www.amazonjp-*.date - urlscan.io](https://urlscan.io/search/#www.amazonjp-*.date)  
<sup id="fn2">[²](#f2)</sup> [3GPP TS 23.040, Technical realization of the Short Message Service (SMS)](https://www.3gpp.org/DynaReport/23040.htm)  
<sup id="fn3">[³](#f3)</sup> [VOICE & MESSAGING TRACK | How SMS Works - Scott Maher (Twilio) - YouTube](https://www.youtube.com/watch?v=d5e3FSy3YLQ#t=37)  
<sup id="fn4">[⁴](#f4)</sup> [Short Message Peer to Peer Protocol Specification v3.4 - SMPP.org](https://smpp.org/SMPP_v3_4_Issue1_2.pdf)  
<sup id="fn5">[⁵](#f5)</sup> [SMPP API - Documentation to Send SMS via SMPP API Online - SMSGlobal](https://www.smsglobal.com/smpp-api/)  
<sup id="fn6">[⁶](#f6)</sup> [RCS Verified Sender - Product Feature Implementation Guideline March 2019 - GSMA](https://www.gsma.com/futurenetworks/wp-content/uploads/2019/03/927_GSMA-RCS-Verified-Sender-report-v5.pdf)
