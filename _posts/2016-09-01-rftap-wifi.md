---
layout: post
title:  "Using RFtap to Detect MAC Spoofing"
categories: blog
---

In this post we'll use [GNU Radio](http://gnuradio.org/)'s [Wi-Fi receiver](https://github.com/bastibl/gr-ieee802-11), [RFtap](/) and [Wireshark](https://www.wireshark.org/) to detect Wi-Fi [MAC spoofing](https://en.wikipedia.org/wiki/MAC_spoofing) by using the unique [carrier offset frequency](https://en.wikipedia.org/wiki/Carrier_frequency_offset) of each Wi-Fi packet.

## What is RFtap?

[RFtap](/) is a simple protocol designed to provide RF (Radio Frequency) metadata about packets, such as:

* Accurate signal and noise power
* Accurate timing and phase information
* Accurate [Carrier](https://en.wikipedia.org/wiki/Carrier_frequency) and [Doppler](https://en.wikipedia.org/wiki/Doppler_effect) frequencies of every packet, and more.

You can think of RFtap as the "glue" between GNU Radio and Wireshark, allowing access to RF metadata from Wireshark or Scapy.

The RFtap protocol is designed to encapsulate any type of packet: Wi-Fi, Bluetooth, or packets from any proprietary protocol. Here we'll encapsulate Wi-Fi packets.

## Prerequisites

Plenty! :smile: You will need a fast Mac or Linux computer (I use Ubuntu 16.04 on Intel i5), a SDR receiver capable of receiving 20MHz bandwidth at a center frequency of ~5GHz such as Ettus [N210](https://www.ettus.com/product/details/UN210-KIT) or [B200](https://www.ettus.com/product/details/USRP-B200mini). Install [GNU Radio from source](http://gnuradio.org/redmine/projects/gnuradio/wiki/InstallingGRFromSource) (I followed [these instructions](https://github.com/gnuradio/pybombs/blob/master/README.md)), install GNU Radio's [Wi-Fi IEEE 802.11 a/g/p transceiver](https://github.com/bastibl/gr-ieee802-11). Verify that you can receive Wi-Fi packets from nearby 5GHz access points by running: `gnuradio-companion gr-ieee-80211/examples/wifi_rx.grc`. See [gr-ieee802-11 README](https://github.com/bastibl/gr-ieee802-11/blob/master/README.md) for troubleshooting. Install [gr-rftap](https://github.com/rftap/gr-rftap). Install [Wireshark](https://www.wireshark.org/) version 2.3 or higher. You can verify that Wireshark supports RFtap by loading this [sample RFtap pcap file](/assets/rftap/rftap_sample.pcap).

[Presentation about GNU Radio IEEE 802.11a/g/p OFDM Receiver](https://www.youtube.com/watch?v=-EvNOvg0USs)

<div class="imgcap">
<img src="/assets/wifi/gr-802-11.png" width="50%">
<div class="thecap">Test setup for demonstrating GNU Radio Wi-Fi transceiver</div>
</div>

Don't have the required hardware? Try the [RFtap RDS tutorial]({% post_url 2016-08-27-decoding-rds-with-rftap %}) instead, using the low-cost [RTL-SDR](http://www.rtl-sdr.com/about-rtl-sdr/) receiver.

## Wi-Fi

When observing Wi-Fi packets in the wild, we can notice that each packet has a slightly different carrier frequency. This [frequency offset](https://en.wikipedia.org/wiki/Carrier_frequency_offset), colloquially named the Doppler shift, is caused by several factors, chief among them:

* Inaccuracy of the crystal oscillator in the Wi-Fi transmitter and receiver
* Velocity of the Wi-Fi transmitter and receiver ([Doppler effect](https://en.wikipedia.org/wiki/Doppler_effect))

As you can probably guess, that's an interesting RF metadata to have, since it can reveal unique properties of the Wi-Fi transmitter.

Despite considerable efforts of manufacturers to keep this frequency error as low as possible (typically less than 20 parts-per-million, or 0.002% of the carrier frequency), even this slight error can wreak havoc on the OFDM receiver. The frequency error typically manifests as a *rotation* effect in the QAM constellation, as seen on the left:

<div class="imgcap">
<img src="/assets/wifi/qam_rotating.gif" width="98%">
<div class="thecap">Left: QAM constellation before correction; Right: QAM after frequency correction and symbol synchronization</div>
</div>

Because of the extreme sensitivity to frequency error, every Wi-Fi receiver in the world implements a complex algorithm to measure and correct for frequency offset. Without this correction, OFDM reception would be practically impossible. So fortunately for us, we don't have to implement anything new to get this offset:

<div class="imgcap">
<img src="/assets/wifi/flow_wifi_sync.png" width="98%">
<div class="thecap">Frequency synchronization in wifi_rx.grc flowgraph</div>
</div>

In the GNU Radio Wi-Fi flowgraph, there are 3 blocks responsible for frequency synchronization:

* "Sync Short" block, making a coarse estimation of the frequency
* "Sync Long" block, making a more accurate estimation of the carrier frequency offset
* "Frame Equalizer" block, making *extra-fine* corrections to the frequency while decoding the symbols of a single Wi-Fi packet

After all this hard work, the carrier frequency estimation for the packet is just thrown away... :cry:

## Tapping the Flowgraph

We'll save this frequency offset estimation using the RFtap Encapsulation block. Since PDU metadata is already passed between Wi-Fi blocks, all we need to do is connect the encapsulation block and a UDP output block:

<div class="imgcap">
<img src="/assets/wifi/wifi_rftap.png">
<div class="thecap">Adding RFtap Encapsulation to Wi-Fi flowgraph</div>
</div>

The frequency offset can now be observed for each Wi-Fi packet in Wireshark:

<div class="imgcap">
<img src="/assets/rftap/rftap_wireshark_tree.png">
<div class="thecap">Adding RFtap Encapsulation to Wi-Fi flowgraph</div>
</div>

## Detecting MAC Spoofing

The frequency offset can be used as a fingerprint for each Wi-Fi packet, differentiating it from other packets regardless of the Wi-Fi MAC address or packet content. We can show the raw frequency offset of each packet in real time:

```
$ tshark -i lo -q -T fields -e wlan.ta -e rftap.freqofs
Capturing on 'Loopback'
00:a0:cc:23:af:4a	837.9016071558
3a:34:52:c4:69:b8	13188.2628193125
3a:34:52:c4:69:b8	15857.2173677385
3a:34:52:c4:69:b8	9895.83786576986
c8:60:00:ba:95:65	3073.38777929544
34:4d:ea:89:75:b2	-6725.89638270438
3a:34:52:c4:69:b8	10878.7475619465
```

Or as a rudimentary plot, again in real-time:

```
$ tshark -i lo -q -l -Y 'wlan.ta' -T fields -e wlan.ta -e rftap.freqofs | \
gawk '{printf("%s %*s\n",$1,($2+300000)/20000,"*")}'

# scale: 1 text column is 20kHz

3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8              *
00:a0:cc:23:af:4a         *
00:a0:cc:23:af:4a       *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8               *
34:4d:ea:89:75:b2    *
34:4d:ea:89:75:b2     *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *

```

Notice that every device has a unique frequency offset identifying it, even at rest (no Doppler effect). If we change our test Wi-Fi device 34:4d:ea:89:75:b2 to a different MAC address (MAC Spoofing), it still retains its unique frequency offset characteristic:

```
3a:34:52:c4:69:b8             *
3a:34:52:c4:69:b8             *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8              *
00:00:00:00:00:01      *
00:00:00:00:00:01     *
3a:34:52:c4:69:b8              *
3a:34:52:c4:69:b8               *
3a:34:52:c4:69:b8              *
```

## Summary

In this post we have shown how to use RFtap to preserve RF metadata from GNU Radio, and how to use RF metadata such as Carrier Offset to uniquely identify devices regardless of the content of their packets. We’ve demonstrated the benefits RFtap can provide in bridging SDR platforms such as GNU radio and network analysis tools such as Wireshark.

## Future Study

This demonstration is using the frequency offset estimation from the **Long Preamble** only. It should be possible to get a **much better** frequency estimation by looking at the entire packet (not just the preamble). There is already a mechanism in frame_equalizer block to track the exact frequency from one symbol to the next. This would be useful to attain Doppler-level accuracies.

It would be interesting to use the frequency offset estimation to measure the speed of vehicles (with on-board Wi-Fi) on the highway (going from +Doppler to -Doppler frequency as they pass the GNU Radio receiver).
