# Transmission and Detection of Silent SMS in Android

<p class="modest" align="left">Jan 17, 2022</p>

---

SMS technology is used for exchanging text messages and for purposes that include SIM management and IoT communication. The technical specifications of SMS technology allow for short messages that do not appear as notifications on the receiving device. These messages, known as "Silent SMS," have the potential to be exploited for covert attacks. How can we determine whether we are under a Silent SMS attack? In this article, we first review the specifications of SMS technology and then send and detect Silent SMS on an Android device.

## Summary

According to SMS specifications, a mobile device receiving a special short message with Type 0 does not notify the user. For example, when we sent Short Message Type 0 using AT commands, no visible notifications appeared on the Android device at the receiving end. However, the device logged a message indicating that it received Short Message Type 0. Therefore, Android users may be able to detect that they are under a Silent SMS attack by monitoring message logs.

## How SMS works

Before going into the details of Silent SMS, let us review the technical specifications of SMS. The short messages sent from a mobile device are first stored in the SMSC (Short Message Service Center), which then delivers them to the destination device. The SMS specification 3GPP TS 23.040 (originally GSM 03.40)<sup id="f1">[¹](#fn1)</sup> defines the message type transferred from the sender's device to the SMSC as "SMS-SUBMIT" and that to be transferred from the SMSC to the receiver's device as "SMS-DELIVER." Figure 1 illustrates the process of short message transfer.

<img src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure1.png">
<p class="modest" align="center">Figure 1: Short message transfer.</p>

The structure of the PDU (Protocol Data Unit) for short message transfer is also defined in 3GPP TS 23.040. This specification defines the PDUs of six message types in the transfer layer. This article describes the PDU structures of SMS-SUBMIT and SMS-DELIVER (shown in Figure 2) as they are related to Silent SMS. TP-MTI (TP-Message-Type-Indicator) indicates the message type of the PDU, while TP-PID (TP-Protocol-Identifier) indicates the protocol to the upper layer. In the case of a normal text message, the TP-DA (TP-Destination-Address) contains the destination phone number, TP-OA (TP-Originating-Address) contains the phone number of origin, and TP-UD (TP-User-Data) contains the message text. Other elements are described in Sections 9.2.2.1 and 9.2.2.2 of the specification.

<img src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure2.png">
<p class="modest" align="center">Figure 2: PDU structures of SMS-SUBMIT and SMS-DELIVER.</p>

## What is Silent SMS

A Silent SMS is a special short message that is not displayed on the screen of the receiving mobile device and does not trigger an acoustic signal. In 3GPP TS 23.040, Silent SMS messages are indicated as Type 0 in the TP-PID. Section 9.2.3.9 of the specification assigns `01000000` as the bit pattern to indicate Short Message Type 0 and defines the handling of the message as follows.

> A short message type 0 indicates that the ME must acknowledge receipt of the short message but shall discard its contents. This means that
>
>  - the MS shall be able to receive the type 0 short message irrespective of whether there is memory available in the (U)SIM or ME or not,
>  - the MS shall not indicate the receipt of the type 0 short message to the user,
>  - the short message shall neither be stored in the (U)SIM nor ME.

The MS (mobile station) in the quoted text is synonymous with UE (user equipment). Therefore, the mobile device that receives this message will not be allowed to notify the user, nor will it be allowed to save the message in the database.

SMS PDUs, where elements other than the TP-PID have been manipulated, may also effectively become Silent SMS. For example, Croft et al. (2007)<sup id="f2">[²](#fn2)</sup> proposed sending Silent SMS using SMPP, which manipulates data coding and WAP push messages. SMPP is a protocol for transferring SMS between External Short Message Entities and the SMSC. Accordingly, the SMPP takes the role of SMS-SUBMIT when sending an SMS. Furthermore, WAP data are included in the TP-UD of the SMS PDU and transferred. Thus, manipulating the TP-DCS (TP-Data-Coding-Scheme) or TP-UD of the SMS PDU may also result in a Silent SMS. This study, however, focuses on Silent SMS with Short Message Type 0 as defined in the specification.

Because the recipient is not notified, a Silent SMS is often used for hidden purposes. Although the use case of Short Message Type 0 is not described in the specification, Section 3.7.7 of *Mobile Messaging Technologies and Services: SMS, EMS and MMS, 2nd Edition* (Gwenaël, 2005)<sup id="f3">[³](#fn3)</sup> describes it as follows.

> The common use case for using this message type is to page a mobile to check if the mobile is active without the user being aware that the mobile station has been paged.

The term "page" in the quoted text refers to the action of a base station requesting a nearby mobile phone to send identification information. It is not necessary for the user to see the short message required for this action. In contrast, since paging reveals the location of the receiving device, German law enforcement agencies use Silent SMS to determine the whereabouts of suspects<sup id="f4">[⁴](#fn4)</sup>. Furthermore, the aforementioned study by Croft et al. suggests that the damage stemming from Silent SMS encompasses DoS attacks as well as economic injury when being charged for receiving SMS. Specifically, Silent SMS has the potential to be exploited for covert attacks. The problem is how these attacks can be detected.

## How to send Silent SMS

To test the detection of Silent SMS, an SMS PDU must be sent as a sample. Unfortunately, there is no API in the Android application framework to directly send SMS PUDs. The package `android.telephony` includes several methods for sending SMS, such as `SmsManager.sendTextMessage()` and `SmsManager.sendDataMessage()`. However, these methods can only manipulate the TP-DA as the destination and TP-UD as the content. In the previous versions of Android, [`SmsManager.sendRawPdu()`](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-1.6_r1/telephony/java/android/telephony/SmsManager.java#224) could be invoked via reflection to send SMS PDUs. In Android 12, this method has been hidden to the internal [`SMSDispatcher.sendRawPdu()`](https://android.googlesource.com/platform/frameworks/opt/telephony/+/refs/heads/android12-release/src/java/com/android/internal/telephony/SMSDispatcher.java#1665), and it no longer accepts the SMS PDU as an argument. To circumvent this restriction, we used an Android device as a USB modem to send SMS PDUs.

We can send SMS PDUs using AT commands by utilizing an Android device as a USB modem; whether the Android device exposes the USB modem interface depends on the model. Tian et al. (2018) showed that the non-rooted Android devices manufactured by Samsung and LG expose the USB modem interface<sup id="f5">[⁵](#fn5)</sup>. This study provides an example of connecting a non-rooted Samsung Galaxy S10 (Android 11) to a MacBook via USB in the MTP mode and then executing AT commands on the exposed USB modem interface `/dev/tty.usbmodemRF8M6███████`. AT commands are used to control modems, and extended commands for controlling SMS have been standardized in 3GPP TS 27.005<sup id="f6">[⁶](#fn6)</sup>. To use AT commands on Galaxy S10, we need to enable Developer options > 3GPP AT commands, as shown in Figure 3.

<p align="center"><img src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure3.png" width="300"></p>
<p class="modest" align="center">Figure 3: Enabling 3GPP AT commands.</p>

To compare the reception of a normal text message with that of a Silent SMS, we sent two types of SMS PDUs, as shown in Figure 4. In this figure, the PDU of a normal text message is shown on the top and that of Short Message Type 0 is shown on the bottom. At the beginning of both PDUs, `00` is added as the SCA (Service Center Address) to indicate the default SMSC, and `01` is set in the TP-MTI to indicate SMS-SUBMIT. The TP-PID is set to `00` for a normal text message, and `40` (bit pattern: `01000000`) for Short Message Type 0. The phone number `+819001234567` set as the TP-DA in the figure is a dummy. The TP-UD is set to `hello` encoded in GSM 7-bit characters, as defined by 3GPP TS 23.038 (originally GSM 03.38)<sup id="f7">[⁷](#fn7)</sup>.

<img src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure4.png">
<p class="modest" align="center">Figure 4: SMS PDUs of a normal text message and Short Message Type 0.</p>

After sending the SMS PDUs, we observed the behavior of the mobile phone that received the PDU of Short Message Type 0. Figure 5 illustrates what happens when the SMS PDUs are sent and received. The SMS PDUs were sent from Galaxy S10 connected through the terminal on the left to the non-rooted Google Pixel 4a (Android 12) on the right. First, we connected to the USB modem interface through the `screen` command and executed `AT` to confirm that AT commands were available. Next, we executed `AT+CMGF=0` to switch the input and output formats to the PDU mode. Thereafter, we executed `AT+CMGS=18` to send the SMS PDUs. The value `18` is the number of octets of the SMS PDU to be sent, excluding SCA. In the case of a normal text message, after the SMS PDU is submitted, `OK` was returned to indicate a successful transmission, and Pixel 4a displayed a notification. In the case of Short Message Type 0, the transmission was successful; however, Pixel 4a did not display a notification. This behavior corresponds to the specification of Silent SMS.

<video controls src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure5.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 5: Sending and receiving SMS PDUs.</p>

## How to detect Silent SMS

Upon receiving Short Message Type 0, Android logs a message. In the project "AIMSICD" for detecting IMSI catchers, the detection of Silent SMS was also discussed<sup id="f8">[⁸](#fn8)</sup>, and a method to detect Short Message Type 0 from the Android log output was adopted. This study reviews Android's log output process with reference to their discussion. If the received SMS-DELIVER is a special short message, Android delegates its handling to the internal [`GsmInboundSmsHandler.disappatchMessageRadioSpecific()`](https://android.googlesource.com/platform/frameworks/opt/telephony/+/refs/heads/android12-release/src/java/com/android/internal/telephony/gsm/GsmInboundSmsHandler.java#155). A part of this method's source code is shown in Figure 6. According to the first comment, some carriers will send visual voicemail as Short Message Type 0; therefore, Android 12 first delegates the received SMS PDU to the visual voicemail process. Thereafter, it outputs a message as a debug-level log, indicating that Short Message Type 0 was received.

```java
protected int dispatchMessageRadioSpecific(SmsMessageBase smsb, @SmsSource int smsSource) {
    SmsMessage sms = (SmsMessage) smsb;

    if (sms.isTypeZero()) {
        // Some carriers will send visual voicemail SMS as type zero.
        int destPort = -1;
        SmsHeader smsHeader = sms.getUserDataHeader();
        if (smsHeader != null && smsHeader.portAddrs != null) {
            // The message was sent to a port.
            destPort = smsHeader.portAddrs.destPort;
        }
        VisualVoicemailSmsFilter
                .filter(mContext, new byte[][]{sms.getPdu()}, SmsConstants.FORMAT_3GPP,
                        destPort, mPhone.getSubId());
        // As per 3GPP TS 23.040 9.2.3.9, Type Zero messages should not be
        // Displayed/Stored/Notified. They should only be acknowledged.
        log("Received short message type 0, Don't display or store it. Send Ack");
        addSmsTypeZeroToMetrics(smsSource);
        return Intents.RESULT_SMS_HANDLED;
    }

```
<p class="modest" align="center">Figure 6: Part of the source code of dispatchMessageRadioSpecific().</p>

While displaying the Android log, we observe the output log message when Short Message Type 0 is received, as shown in Figure 7. The top of the terminal connects to Galaxy S10 (the sender) and the bottom connects to Pixel 4a (the receiver). When displaying Pixel 4a logs, we set the `logcat` command with the option `-b radio`, to request logs related to radio and telephony. Moreover, two filters `GsmInboundSmsHandler:D *:S` were set to display only the target logs. After sending Short Message Type 0 from Galaxy S10, although Pixel 4a did not display a receipt notification, it outputted a log message indicating that it received Short Message Type 0. In Figure 7, this is highlighted in red text. Therefore, we know that Short Message Type 0 was received.

<video controls src="/assets/2022/transmission_and_detection_of_silent_sms_in_android/23_figure7.mp4" type="video/mp4"></video>
<p class="modest" align="center">Figure 7: Outputting a log message.</p>

## Conclusion and Future Work

Android is designed to output a log message when a Short Message Type 0 is received. By monitoring this log, we may be able to detect attacks that exploit Silent SMS. However, Silent SMS can also be achieved using methods other than Short Message Type 0. Therefore, further research should investigate whether these methods can be proven and detected. Furthermore, since this study solely focuses on Android, future work should investigate the detection of Silent SMS in iOS. If a Silent SMS attack is detected but the carrier does not provide any countermeasure, a possible temporary measure is to set the mobile device to airplane mode or remove the SIM card. This measure, however, disables telephony as well as SMS. Therefore, future studies are required to determine a method for mobile devices to reject only Silent SMS.

---

<sup id="fn1">[¹](#f1)</sup> [3GPP TS 23.040, Technical realization of the Short Message Service (SMS)](https://www.3gpp.org/DynaReport/23040.htm)  
<sup id="fn2">[²](#f2)</sup> [Croft, Neil J., and Martin S. Olivier. "A Silent SMS Denial of Service (DoS) Attack"](https://mo.co.za/open/silentdos.pdf)  
<sup id="fn3">[³](#f3)</sup> [Mobile Messaging Technologies and Services: SMS, EMS and MMS, 2nd Edition - Wiley](https://www.wiley.com/en-us/Mobile+Messaging+Technologies+and+Services%3A+SMS%2C+EMS+and+MMS%2C+2nd+Edition-p-9780470014516)  
<sup id="fn4">[⁴](#f4)</sup> [Zoll, BKA und Verfassungsschutz verschickten 2010 über 440.000 "stille SMS" - heise online](https://www.heise.de/newsticker/meldung/Zoll-BKA-und-Verfassungsschutz-verschickten-2010-ueber-440-000-stille-SMS-1394593.html)  
<sup id="fn5">[⁵](#f5)</sup> [Tian, Dave Jing, et al. "ATtention Spanned: Comprehensive Vulnerability Analysis of AT Commands Within the Android Ecosystem"](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-tian.pdf)  
<sup id="fn6">[⁶](#f6)</sup> [3GPP TS 27.005, Use of Data Terminal Equipment - Data Circuit terminating Equipment (DTE - DCE) interface for Short Message Service (SMS) and Cell Broadcast Service (CBS)](https://www.3gpp.org/DynaReport/27005.htm)  
<sup id="fn7">[⁷](#f7)</sup> [3GPP TS 23.038, Alphabets and language-specific information](https://www.3gpp.org/DynaReport/23038.htm)  
<sup id="fn8">[⁸](#f8)</sup> [Detection of Silent (Stealth) SMS Type 0 [$115 awarded] · Issue #69 · CellularPrivacy/Android-IMSI-Catcher-Detector - GitHub](https://github.com/CellularPrivacy/Android-IMSI-Catcher-Detector/issues/69)
