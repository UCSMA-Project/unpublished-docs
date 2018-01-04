In this document I will explain the topology I used to conduct experiments, as well as the procedures to setup this topology.
# Desired Experimental Topology
![alt-text](https://docs.google.com/drawings/d/1dafM3TLEyatLZ2xl3SZObij7oYd1kKr1YVpbK1QFoe8/pub?w=1155&h=739 "Desired Experimental Topology")  
The desired topology is:
1. Mid node is within signal range of both edge nodes.
2. Both edge nodes are within signal range of mide node.
3. Both edge nodes are beyond each other's signal range.

And the desired topology will imply the following behaviors:
1. When either of the edge node is transmitting, the mid node will sense a busy channel.
2. When mid node is transmitting, both edge nodes will sense a busy channel.
3. Both edge nodes can transmit freely without interfering each other.

However, creating such a topology in a real world scenario truns out to be quiet challenging, and this is because:
1. Wireless adapters are designed to be adaptvie to the surrounding environment. On my AR9331 development board, there is a calibration process that will adjust the noise floor. However, this calibration process can adversely affect my experiments. For example, if both edge nodes are transmitting, the mid node will find it surrounded by a noisy environment, the calibration process will adjust the noise floor so that the mid node will treat the mixed signal as background noise, and think the channel is idle.
2. For edge nodes to not interfere with each other, it either requires a long distance between the two nodes or a really low TX power. To make matters worse, the calibration process will dynamically adjust the noise floor, making changing distance effortless (e.g. at the same distance, different calibrated noise floors will cause different behavior, sometimes the edge nodes can sense each other, and sometimes not).
3. All three nodes are subject to interference from external wireless signals and atmosphere noise. Many times the auto-calibrated noise floor is too low (too sensitive) and the nodes can sense a lot of external signals.

# ATH9K Calibration Process
In order to setup the desired topoloy, I need to figure out the calibration process first, as this can potentially solve:
1. Interference from external source. If I can turn off the auto-calibration and manually set the noise floor, I can set it to a relative high value (less sensitive) so that all three nodes will just ignore the external signals.
2. Undesired transmission from mid node. I can set the noise floor to a certain value so that the mid node won't treat the mixed signal from edge nodes as background noise, therefore preventing the undesired transmission from mid node when edge nodes are transmitting.

After carefully examing the calibration related driver code, I obtain the following conclusions:
1. Noise floor calibration is done periodically or when a beacon stuck event occurs.
2. The actually calibration is done internally in the hardware, but it does have an option to tell the hardware whether to update calibrated noise floor automatically or just do the calibration without updating noise floor.
3. The ath9k driver can fetch calibrated noise floor value from the hardware and apply some filters (e.g. set a maximum/minimum allowed noise floor), then load the filtered value to the hardware.

The most interesting and promising part is that the ath9k driver has the ability to load a noise floor value into the hardware, and this is what I am looking for.
## Load Noise Floor into Hardware
The driver code of this part is in function ```ath9k_hw_loadnf``` from ```calib.c```, the sketch of this function is as follows:
1. Read calibrated noise floor (already filtered) from previous calibration process, if no data avaiable, load the default noise floor.
2. Time the noise floor value by 2, and write it to the last 9 bits of register ```AR_PHY_CCA_0``` (AR9331 has only one chain, for other adapters with multiple chains, writting ```AR_PHY_CCA_1``` and ```AR_PHY_CCA_2``` may be required).
3. Load software filtered noise floor value into baseband internal minCCApower variable. This is done by writting some bits in ```AR_PHY_AGC_CONTROL``` register, and it will make the value written in step 2 effective.
4. Wait until the load in step 3 is completed. This is done by checking whether the ```AR_PHY_AGC_CONTROL_NF``` bit in ```AR_PHY_AGC_CONTROL``` register equals to 0.
5. Write -50 * 2 to the last 9 bits of register ```AR_PHY_CCA_0```. According to the driver code comment, this value (-50) will be the initial (and max) noise floor of next calibration the baseband does. Note here the driver only writes the ```AR_PHY_CCA_0``` register but not the ```AR_PHY_AGC_CONTROL``` register, so the noise floor value -50 won't become effective.

During my research, it took me very long time to finally fully understand this part of driver code. In the [AR9331 datasheet](https://www.openhacks.com/uploadsproductos/ar9331_datasheet.pdf), it doesn't even mention anything about the ```AR_PHY_AGC_CONTROL``` or ```AR_PHY_CCA_0``` register, so I had a very hard time figuring out what these registers control. I can only guess and infer the meaning of the registers from ath9k driver code.

Luckily, now I have full understanding of how the ath9k driver loads a noise floor value into the hardware, I can just do a simple hack to enable manual load of noise floor. This [patch](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/791d942f048f0291eea2f0c0f9f45e3f86d11edd) adds an override noise floor option to the ath9k driver, and I can simply set noise floor by executing
```
echo [noise floor value] > /sys/kernel/debug/ieee80211/phy0/ath9k/nf_override
```
## Experiment: Manual Adjustment of Noise Floor
With the ability of manually set noise floor, I should be able to create the desired topology easily. To make things easy to manage, I did my experiment in a metal box. I took the antenna of both edge nodes off, so that they would have a reduced TX/RX ability (i.e. TX power is less because of no antenna, RX sensitivity is less due to no antenna), therefore having less possibility of interfering with each other. I kept the antenna of the mid node on, in that way it would have a strong TX power to compensate the reduced RX sensitivity on edge nodes (so that edge nodes would still sense the signal from mid node), and also a more sensitive RX to compensate the reduced TX power on edge nodes (so that mid node would still sense the signal from either edge node). I did the following to setup the experiment environment:
1. Place all three nodes in a line, with about 15cm distance between edge nodes and mid node.
2. Adjust TX power on edge nodes as well as the antenna direction on the mid node, so that the mid node senses approximately equal signal strength from both edge nodes.
3. Increase noise floor on edge nodes, until they cannot sense each other at all, confirm this by checking if an edge node can use tcpdump to capture frames sent by another edge node, as well as checking if the CRC ERR counter increases while another edge node is transmitting (sometimes one node may not capture any frames sent by another node, but the CRC ERR counter does increase, indicating partial frames have been detected).
4. Increase noise floor on mid node to reduce external interference, while making sure that it can still capture frames sent by either edge node.

After setting up the environment, I did the following experiments (the parameters were CW = 1, AIFS = 3, SIFS = 8us, slottime = 20us):
1. __single node TX__. I let one node transmit as fast as possible, while the other two nodes remain silent. I did this to all three nodes, and they all achieved the similar rate. This experiment showed that the external interference is neglectable, because the mid node has antenna on while edge nodes have antenna off, if external interference is significant, I should have noticed a obvious drop in TX rate from mid node.
2. __both edge nodes TX__. I let both edge nodes transmit as fast as possible, and they both achieved the same rate as in experiment 1. This indicates that the edge nodes indeed don't interfere with each other at all.
3. __mid node and one edge node TX__. I let the mid node and one of the edge nodes transmit as fast as possible, and they achieved a similar rate (but not exactly half of the rate in experiment 1). This indicates that these two nodes approximately share the channel 50:50.
4. __all three nodes TX__. I let all three nodes transmit as fast as possible. This is where things went unexpected, I observed a higher TX rate than experiment 3 for all three nodes.

The experiment 4 in the above experiments has unexpected result, which indicates that there is something I forgot to take into consideration. After a long time of researching, I finally found the reason that can explain this unexpected behavior.
## Mode of Clear Channel Assessment
The wireless adapter use CCA (Clear Channel Assessment) to determine if the wireless channel is busy. And it appears that there exist three different modes of CCA:
1. __CCA-ED (Energy Detection)__. In this mode, if the signal strength on the channel is beyond a certain threshold, the adapter thinks the channel is busy. The recommended threshold for this mode is -62dBm. Note that the signal can be either decodable or undecodable.
2. __CCA-CS (Carrier Sense)__. In this mode, the adapter is contantly looking for 802.11 preamble and 1Mbps header info when idle. If the adapter detects a decodable 802.11 signal, it will think the channel is busy. The recommended threshold for this mode is -82dBm.
3. __Combined Mode__. Both CCA-ED and CCA-CS are used in determining whether the channel is busy.

From the previous experiment results, I can infer that my AR9331 development boards are using combined CCA mode. In experiment 3, when mid node and one of the edge nodes are transmitting, the mid node can receive decodable 802.11 preamble and correctly sense busy channel. However, in experiment 4, when all three nodes are transmitting, the mixed signal from the two edge nodes makes mid node no longer able to decode the 802.11 preamble, and at this point the CCA-ED mode is used to determine whether the channel is busy. Since the threshold in CCA-ED mode is higher than CCA-CS mode, the adapter thinks the channel is idle and starts to transmit when it should not.
## Adjustment of CCA Threshold
To solve the undesired behavior, I need to figure out a way to adjust the CCA-ED threshold, so that when Energy Detection is engaged, the mid node will still consider the mixed signal as busy channel. However, this again turns out to be a chanellenging task as there is no mention about CCA threshold at all in neither AR9331 datasheet or the ath9k driver.

This is a ```AR_PHY_CCA_THRESH62``` register defined in ```ar9003_phy.h``` but the whole ath9k driver doesn't reference it at all except in ```eeprom_def.c```, which again turned out to be a dead end as this part of driver code is never executed. Luckily, I finnally found what I need in [Qualcomm HAL of AR9003](https://github.com/qca/qcamain_open_hal_public/tree/master/hal/ar9300). There is a function ```ar9300_set_cca_threshold``` that sets the CCA threshold for AR9300 series wireless chip. It turns out that it is not possible to select the CCA mode for ath9k device (some other wireless chips may allow user-chosen CCA mode, e.g. [CC2520](https://e2e.ti.com/support/wireless_connectivity/zigbee_6lowpan_802-15-4_mac/f/158/p/230561/809685#809685)), but adjusting the CCA threshold on ath9k will both adjust the threshold for CCA-ED and CCA-CS. I don't know how exactly the CCA threshold in ath9k affects the CCA-ED and CCA-CS thresholds, but lowering it will make the adapter more sensitive to weak signals and treat them as busy channel.

Understanding this, I made a [patch](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/619cf9f508359bca5fdb9b2052ba007982ed48c4) to write and read the CCA values in the hardware register. The register that stores the max CCA and min CCA power is called ```AR_PHY_CCA_0```, and its memory address is 0x9e1c. It has the following structure:
1. Bit 0:8 (mask 0x1ff), 9 bits in total. These bits store the doubled to-be-loaded noise floor value (i.e., if the to-be-loaded noise floor is -100dBm, write -100 \* 2 here), certain bits in  ```AR_PHY_AGC_CONTROL``` register need to be written in order to actually load this noise floor value in to baseband.
2. Bit 12:18 (mask 0x7f000), 7 bits in total. These bits store the maxCCApower value in the baseband, and control the threshold that CCA uses.
3. Bit 20:28 (mask 0x1ff00000), 9 bits in total. These bits store the minCCApower (a.k.a noise floor) value in the baseband. Note that directly writing these bits won't change the baseband's noise floor, the new value has to be written to structure 1 and then loaded by writting ```AR_PHY_AGC_CONTROL``` register.

As far as I understand, the minCCApower and maxCCApower have the following effects to the hardware and driver:
1. __minCCApower__, a.k.a noise floor. The hardware will use this value as a reference point to detect a frame and calculate its RSSI. For example, if a received frame has RSSI of -60dBm when minCCApower value is -100, then the same frame will have RSSI of -70dBm if the minCCApower value is adjusted to -90. Frame and signal with too low RSSI will be ignored by the hardware, so be careful when adjusting the minCCApower, if this value is set too high (e.g. -50), then it will cause the hardware to be deaf and ignore everything around it.
2. __maxCCApower__. The hardware will use this value in CCA (Clear Channel Assessment). In original ath9k driver this value is never modified, and the default value is depend on the eeprom of the hardware (for my AR9331 development board it is -95). Note that the maxCCApower value is based on calucated RSSI of a signal, which means the same signal can make CCA thinks the channel is idle or busy, depending on how the minCCApower value is set. For example, originally a signal causes the CCA to think the channel is busy, now keep maxCCApower fixed and gradually increase minCCApower (noise floor), to some point the same signal will no longer make the CCA thinks the channel is busy.

In a summary, increasing minCCApower (noise floor) will make surrounding signals weaker in the eye of the adapter, and to keep the same signal treated as busy channel, maxCCApower needs to be lowered correspondingly. It was quiet hard for me to finally figure all this out, as there was no existing doucments mentioning anything about this, and no description of the related registers in AR9331 datasheet.
## Setup Desired Topology
With full understanding of the minCCApower and maxCCApower mechanisms in ath9k, I can finally setup the desired topology easily by the following steps:
1. Place all three nodes in a line, with about 15cm distance between edge nodes and mid node.
2. Adjust TX power on edge nodes as well as the antenna direction on the mid node, so that the mid node senses approximately equal signal strength from both edge nodes (montior singal strength by using tcpdump on monitor interface, it will display RSSI value for each captured packet).
3. Increase noise floor on edge nodes, until they cannot sense each other at all. Confirm this by checking if an edge node can use tcpdump to capture frames sent by another edge node, as well as checking if the CRC ERR counter increases while another edge node is transmitting (sometimes one node may not capture any frames sent by another node, but the CRC ERR counter does increase, indicating partial frames have been detected).
4. Increase noise floor on mid node to reduce external interference, while making sure that it can still capture frames sent by either edge node.
5. Decrease maxCCApower on all three nodes accordingly, so that they won't ignore each other's signal in CCA-ED mode.
6. Further decrease maxCCApower on both edge nodes, so that in the competing round after an unlocking happens, the edge nodes won't violate the desired topology.

Adding step 5 to my original steps eliminated the unexpected behavior in previous experiment 4, and verified that I have created the desired topology. After that I did another set of [experiments](https://gist.github.com/JackWindows/708984463097954ed9e80b90bc1e89a7) in the desired topology for further verification.

Adding step 6 is to elimiate the violation behavior in the competing round after an unlocking happens. Originally, after an unlocking happens, in the first round of competing, the edge nodes sometimes violate the topology and start to transmit even though the mid node has acquired the channel (started transmission). By further decreasing maxCCApower on edge nodes, this violation behavior can be elminated.