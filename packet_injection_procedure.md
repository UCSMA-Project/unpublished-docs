During my reasearch, one important problem is to figure out a way to make the wireless adapter transmit as fast as possible. In this document I will explain how to accompish that using a packet injection program.
# Wireless Packet Injection
Wireless packet injection allows users to construct custom-made packets and send them via wireless adapters. Luckily, my AR9331 development boards and ath9k driver support packet injection, which enables me to send custom-made packets as well as partially control how fast the kernel feeds packets to the wireless adapter. Below is a diagram illustrating the workflow.  
![alt-text](https://docs.google.com/drawings/d/19NdvtEkat8W2Z1aSBoj9iikaNtXVfltft_-0MEvtG6Y/pub?w=1055&h=687 "Packet Injection Workflow")  
There are multiple stages in the workflow:
1. User space program construct a packet, fill in the content and radiotap header, then pass it to the mac80211 abstract layer through ```libpcap```.
2. The mac80211 abstract layer parse the radiotap header of the packet, use the parsed infromation to set transmission flags of the packet, then pass it to the ath9k driver.
3. The ath9k driver instantiates a number of functions defined in the mac80211 abstract layer, it assembles the frame using the set control flags and put the assebmled frame into hardware buffer.
4. The wireless adapter runs CSMA protocol and transmits the frame when appropriate.

I will explain the above four steps in details.
## User Space Packet Injection Program
I wrote a [packet injection program](https://github.com/JackWindows/packetspammer) based by [this](https://wireless.wiki.kernel.org/en/users/documentation/packetspammer). This program has the following functionalities:
1. Construct a packet with radiotap header and pass it to OS kernel by calling function ```pcap_inject``` in ```libpcap```.
2. Set the delay between passing two packets to OS kernel. Setting the delay to 0 essentially tells the hardware to transmit as fast as possible.
3. Calculate the actually rate that it feeds packets to the OS kernel. This is done by using ```libpthread```.

Note that generally function ```pcap_inject``` is non-blocking (i.e. it returns instantly instead of waiting until the packet is actually been transmitted), however, obviously the hardware cannot transmit frames as fast as ```pcap_inject``` feeds packets and the buffer space is not infinite. Therefore, when the buffer space in mac80211 abstract layer is full, function ```pcap_inject``` will become blocking, i.e. it will return once next buffer space in mac80211 abstract layer is available. In another word, when buffer space mac80211 abstract layer is full, the packet injection program can only pass packets to the OS kernel as fast as the adpater transmits frames, therefore, having functionality 3 enables me to monitor at which rate the hardware is transmitting frames.
### Radiotap Header
The radiotap header in the constructed packets is a pseudo header (i.e. it is not a part of the actual frames). It contains some control information (e.g. transmitting rate/channel/modulation/whether packet has FCS, Frame Check Sequence), the mac80211 abstract layer will later parse these information and set transmission control flags accordingly.

Note that it is up to the mac80211 abstract layer's decision to how to use the information stored in radiotap header, in extreme case it will ignore all information in the radiotap header and just use the default parameters or whatever values that are already in the drvier.
## MAC80211 Abstract Layer
In the mac80211 abstract layer, the radiotap header of injected packet gets parsed and used to set transmission control flags for that packet. According to the OpenWrt 14.04 mac80211 abstract layer source code (```ieee80211_parse_tx_radiotap``` in ```mac80211/tx.c```), it will parse the FCS (Frame Check Sequence), DONT_ENCRYPT, DONT_FRAG and NO_ACK flags in the radiotap header, all other control information is discarded.

Apart from parsing radiotap header, the mac80211 abstract layer also acts as a bridge between the ath9k driver and user space wireless utilities such as ```iw```. This part is called ```nl80211```, it is an kernel API exposed to user space programs to communicate with the wireless driver and set the adpater's working parameters such as channel/TX power.
## ATH9K Wireless Driver
The ath9k driver works closely with the mac80211 abstract layer, it instantiates a number of functions to achieve the required functionalities, e.g. setting hardware registers accordingly when mac80211 abstract layer requests a channel change.

In my research, I set my wireless adapters to operate in sole monitor mode, however, the mac80211 abstract layer and ath9k driver have poor support for pure monitor mode (normally monitor mode is set on a slave interface rather than the master interface). In order for my packet injection program to better cooperate with the ath9k driver, I made the following enhancement to both mac80211 abstract layer and ath9k driver:
### Choose Transmitting Rate for Injected Packets
In the mac80211 abstract layer shipped with OpenWrt 14.04, unfortunately it does not parse the transmitting rate flag in radiotap header. Therefore, the mac80211 abstract layer does not set transmitting rate parameter in transmission control flags, and the ath9k driver will select the first available transmitting rate supported by the wireless adapter.

In order to use the desired transmitting rate for injected packets, I need to modify the ath9k drvier code and reorder the list of supported data rate (so that the desired rate is the first element in the list). See [this patch](https://github.com/JackWindows/OpenWRT-14.07-JS9331/blob/a70511d46bc2fe2d44744e1957106d017c0b9294/package/kernel/mac80211/patches/931-bitrate_order.patch) for detailed code.
### Improve Support for Pure Monitor Mode
Previously I mentioned the default mac80211 abstract layer has poor support for pure monitor mode, a lot of the its user space API (```nl80211``` and ```cfg80211```) is broken when the wireless adapter operates in monitor mode only. Therefore, I made several patches to improve support for pure monitor mode:
1. Allow setting txq_params (TX queue parameters, e.g. min/max contention window, AIFS, burst value) for monitor mode interfaces. This involves two patches, see [1](https://github.com/JackWindows/OpenWRT-14.07-JS9331/blob/master/package/kernel/mac80211/patches/934-allow_modification_of_txq_params_in_mesh_and_monitor_mode.patch) and [2](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/d3c375feb36a0c5cac86de794e15c931fb3cbf4e).
2. Allow setting contention winodw value to 0. In default ath9k driver, the minimum value of contention window is 1, however, the wireless adapter actually supports contention window of 0, this patch enables me to conduct some hehavior experiments of the hardware. The code of this patch is [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/blob/master/package/kernel/mac80211/patches/948-allow_zero_cwmin_cwmax.patch).
3. Allow adjusting TX power in pure monitor mode. In original mac80211 abstract layer, it is impossible to adjust TX power if the wireless adapter has only monitor mode interfaces. This patch fixes that, the code is [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/122b2cc43d21b22d70548858fdc93c6196070cdf).
### Add option to disable EIFS (Extended Inter Frame Spacing)
In 802.11 protocol, if a wireless adapter detects 802.11 wireless signal but failed to decode the content of that frame, it will switch its post-frame backoff method from regular AIFS to EIFS. Normally EIFS is much longer than AIFS (typical EIFS=362us while AIFS=68us), which will give a huge distanvange for the mid node to compete the channel in my experiments. To override this behavior, I added an optional debug parameter in the ath9k driver that if enabled, will set EIFS to be equal to AIFS. The code of this patch is [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/blob/master/package/kernel/mac80211/patches/949-optional_eifs_equal_to_aifs.patch), note that after enabling this debug paramter, txq_params has to be adjusted by using ```iw``` tool before the override takes effect.
## Wireless Adapter
All injected packets will finally be put into the hardware TX buffer and transmitted by the wireless adapter. The wireless adapter has its onboard logical of running CSMA and transmitting frames when channel available. For a detailed explanation, see [this section](https://gist.github.com/JackWindows/c7921b76ebc6f3f960b3a2a867435637).

Note that it is important that the injected packets have the NO_ACK flag set, otherwise the adapter will expect an ACK for each injected packet and block further transmission until ACK is received or timeout occurs.