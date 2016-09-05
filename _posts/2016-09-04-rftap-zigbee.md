---
layout: post
title:  "New RFtap Zigbee demo added"
categories: blog
---

## New Zigbee Demo

Zigbee demo added to [gr-rftap](https://github.com/rftap/gr-rftap). See gr-rftap examples directory. Notice the [RFtap](/) "Signal Quality" metric available in Wireshark for every packet:

<div class="imgcap">
<img src="/assets/zigbee/zigbee_rftap_wireshark.png">
<div class="thecap">RFtap Zigbee demo</div>
</div>

In order to achieve this result, available in the examples/zigbee_rftap.grc flowgraph, we add two blocks:

* RFtap Encapsulation
* LQI to qual

<div class="imgcap">
<img src="/assets/zigbee/zigbee_rftap_flowgraph.png">
<div class="thecap">Zigbee flowgraph zoom - RFtap section</div>
</div>

In the RFtap encapsulation block, we specify a Custom Data Link Type of 195 (Zigbee), as per this [linktype list](http://www.tcpdump.org/linktypes.html).

<div class="imgcap">
<img src="/assets/zigbee/rftap_block.png">
<div class="thecap">RFtap block properties</div>
</div>

As for the Signal Quality property, we use the Link Quality Indicator (LQI) available from 802.15.4 block, and convert it to RFtap signal quality (qual) field using an embedded python block:

<div class="imgcap">
<img src="/assets/zigbee/lqi_block.png">
<div class="thecap">Embedded python block properties - "LQI to qual"</div>
</div>

The embedded code:

```python
import numpy as np
from gnuradio import gr
import pmt

class blk(gr.basic_block):
    """Convert Zigbee Link Quality Indicator (LQI) (0..255) 
       to RFtap signal quality field (qual) (0.0..1.0)"""

    def __init__(self):
        gr.basic_block.__init__(
            self,
            name='LQI to qual',   # will show up in GRC
            in_sig=[],
            out_sig=[]
        )
        self.message_port_register_in(pmt.intern('in'))
        self.set_msg_handler(pmt.intern('in'), self.handle_msg)
        self.message_port_register_out(pmt.intern('out'))

    def handle_msg(self, pdu):
        meta, data = pmt.to_python(pdu)
        meta['qual'] = meta['lqi'] / 255.0
        pduout = pmt.cons(pmt.to_pmt(meta), pmt.to_pmt(data))
        self.message_port_pub(pmt.intern('out'), pduout)
```

The demo uses the GNU Radio [802.15.4 Zigbee module](https://www.wime-project.net/features/#zigbee), part of the [WiME project](https://www.wime-project.net/):

<div class="imgcap">
<img src="/assets/zigbee/zigbee_rx_wime.png">
<div class="thecap">IEEE 802.15.4 ZigBee WiME project</div>
</div>

## What is RFtap?

[RFtap](/) is a simple protocol designed to provide RF (Radio Frequency) metadata about packets, such as:

* Accurate signal and noise power
* Accurate timing and phase information
* Accurate [Carrier](https://en.wikipedia.org/wiki/Carrier_frequency) and [Doppler](https://en.wikipedia.org/wiki/Doppler_effect) frequencies of every packet, and more.

You can think of RFtap as the "glue" between GNU Radio and Wireshark, allowing access to RF metadata from Wireshark or Scapy.

The RFtap protocol is designed to encapsulate any type of packet: Wi-Fi, Bluetooth, or packets from any proprietary protocol.
