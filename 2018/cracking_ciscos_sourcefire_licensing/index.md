---
layout: post
author: Jose Krause
title: "Cracking Cisco’s Sourcefire licensing system"
subtitle: "Free the Kraken!"
date: 2018-04-02T15:00:00+02:00
tags: ["reversing", "cisco", "security", "golang"]
---

Cisco’s Sourcefire system is the IDS/IPS solution offered by this company after the acquisition of Sourcefire, including its network anomaly detection engine, Snort. This IPS solution is one of the most powerful systems available on the market.

The system is composed mainly by two appliances:

* The **sensor** –FirePOWER–, is the IPS itself with Snort, the RNA –Real Network Awareness– engine, nmap, the signature database and all the stuff that makes sense on an IPS. This appliance is mainly physical but Cisco offers also a virtual appliance option available on the customer support portal.

* The **manager** –FireSIGHT Management center (FSM)–, is the central administration console, one FSM can have attached multiple sensors, and all the configuration is done here, so as policy creation, firewall rules, object setup, rule edition, etc. Once configured or modified some policy the whole config/rule/stuff package is deployed to the paired sensors. This element can be run as a virtual appliance available on the Cisco customer support portal.

The main problem of Cisco’s Sourcefire system is that the hardware is completely useless without a valid license. After buying a sensor on Ebay or scavenging one from a death project or whatever, a license is still needed to make them to work, and yes, these licenses are not exactly cheap...

The laboratory setup used for the paper uses this setup:

* Virtual FSM on version 5.4.1.9
* Physical sensor 3D2000 on version 5.4.0.9
* Physical sensor 3D7110 on version 5.4.0.9

But the bypass techniques exposed in the paper are also applicable to the latest versions of Sourcefire sensors and FSMs – Tested on FSM version 6-.

According to Cisco, neither its ASA nor the new Firepower Threat Defense (FTD) appliances are susceptible to the demonstrated license bypass.  However, I am not able to confirm or deny this as I haven’t had the chance to test those systems.

_**Paper at the end**_

## Disclosure timeline
* **_02/21/2018_**: Reported to Cisco Talos Team under the address (research@sourcefire.com). It is available on [www.talosintelligence.com/about](https://www.talosintelligence.com/about)
    * No response.
* **_03/07/2018_**: Sent email reminder.
    * No response.
* **_03/15/2018_**: Sent email reminder.
    * No response.
* **_03/15/2018_**: Announced the public disclosure of the paper on [Twitter](https://twitter.com/bitsniper/status/974231110132658178).
* **_03/15/2018_**: Response from Omar Santos (Cyber security principal engineer at Cisco's PSIRT).
* **_03/15/2018_**: Email sent to Cisco's PSIRT as requested by Omar.
* **_03/16/2018_**: ACK from Cisco's PSIRT.
* **_03/16/2018_**: Received an email from Henry Peltokangas (Product Manager at Cisco working on the Firepower software) apologizing for the non-response, arguing that the research@sourcefire.com mailer is no longer monitored and asking for delaying the publication to the end of the month and modify a paragraph on the papers introduction.
*  **_03/16/2018_**: I agreed with the terms and begin to work with Henry on the disclosure process.
*  **_03/21/2018_**: Changed the paragraph by a more exact one as asked by Henry.
*  **_03/21/2018_**: Henry sent me a list of affected devices.
*  **_04/02/2018_**: Full disclosure.


## Affected versions
According to Cisco, these versions are susceptible to apply this cracking techniques.

* Firepower 8120, 8130, 8140 
* Firepower 8250, 8260, 8270, 8290
* Firepower 8350, 8360, 8370, 8390
* AMP8050 AMP8150, 8350, 8360, 8370, 8390 
* Firepower 7050 
* Firepower 7010, 7020, 7030 
* Firepower 7110, 7115 7120, 7125 AMP7150

## Links
* [Cracking Cisco's Sourcefire licensing System paper](/files/cracking_sf_license_system.pdf)
* [Paper code](https://github.com/hcninja/sflicense)
