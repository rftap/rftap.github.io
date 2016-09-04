---
layout: post
title:  "New RFtap Zigbee demo added"
categories: blog
---

## New Zigbee Demo

Zigbee demo added to [gr-rftap](https://github.com/rftap/gr-rftap). See gr-rftap examples directory. Notice the [RFtap](/) "Link Quality" metric available in Wireshark for every packet:

<div class="imgcap">
<img src="/assets/zigbee/zigbee_rftap.png">
<div class="thecap">RFtap Zigbee demo</div>
</div>

The demo uses the GNU Radio [802.15.4 Zigbee module](https://www.wime-project.net/features/#zigbee), part of the [WiME project](https://www.wime-project.net/):

<div class="imgcap">
<img src="/assets/zigbee/zigbee_rx.png">
<div class="thecap">IEEE 802.15.4 ZigBee WiME project</div>
</div>

## What is RFtap?

[RFtap](/) is a simple protocol designed to provide RF (Radio Frequency) metadata about packets, such as:

* Accurate signal and noise power
* Accurate timing and phase information
* Accurate [Carrier](https://en.wikipedia.org/wiki/Carrier_frequency) and [Doppler](https://en.wikipedia.org/wiki/Doppler_effect) frequencies of every packet, and more.

You can think of RFtap as the "glue" between GNU Radio and Wireshark, allowing access to RF metadata from Wireshark or Scapy.

The RFtap protocol is designed to encapsulate any type of packet: Wi-Fi, Bluetooth, or packets from any proprietary protocol.
