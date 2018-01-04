In this document I will explain the original TX path in ath9k driver as well as how I modified it to help controling TX status.  
__Note 1__: this doucment is intended for the ath9k driver from ```compat-wireless-2014-05-22``` with all the patches from OpenWrt 14.07(git://github.com/openwrt/openwrt.git).  
__Note 2__: the hardware I use is an AR9331 based development board.
# Original ath9k TX path
There are already various documents out there explaining the ath9k TX path (e.g. http://www.campsmur.cat/files/mac80211_intro.pdf), but they are only high level explanation without too deep dive in to the hardware queue component. In my research project I mainly interested in figuring out how the ath9k driver communicates with the adapter to transmit frames, so my analysis will focus on the low-level part about how exactly the ath9k driver prepares frames for the adapter and initiates transmission.

My device (AR9331) has EDMA (Enhanced Direct Memory Access, see explanation here http://nearhop.blogspot.ca/2013/10/network-driver-rx-descriptors-ath9k-as.html), so I will only focus on the EDMA code path in the following code analysis.
## Original ath9k TX path Flowchart
![alt text](https://docs.google.com/drawings/d/11vWaQMTt43sf719bhqPfPv_sHeG-_wEk7dGfnPFHs7E/pub?w=960&h=720 "ATH9K Original TX path")
## TX start procedure
__TXFIFO queue__: For each hardware queue (in AR9331 there are ten hardware queues), there is a TXFIFO queue associated with it. According to the ath9k driver, TXFIFO queue is a circular array of length 8, and each of its element is a list of frames to be transmitted. For non-EDMA device, there won't be this TXFIFO queue, but only a single list of frames associated with each hardware queue.  
Conjecture: It seems EDMA device has the ability of memorizing 8 different start points of frames, while non-EDMA device can only memorize 1 start point of frame.

At the end of ath9k TX path, all new frames have been synced from the OS memory to the DMA of the wireless chip (which means at this point both OS kernel and the adapter should have direct addess to the frame data). At this point (in function ```ath_tx_txqaddbuf```), the ath9k driver will give instructions to the adapter based on current status of the TXFIFO queue:
### TXFIFO queue has at least one empty slot
This indicates the adapter has space to memorize the start point of this new list of frames. The ath9k driver will put the new list of frames into the empty slot of TXFIFO queue, and write the address of the first frame buffer in this new list of frames to AR_QTXDP (the queue TX descriptor pointer) register of corresponding hardware queue.

When writting the register, the adapter can be in different status:  
1. All 8 slots of TXFIFO queue are empty before writting the register, and this is the very frist list of frames the adapter needs to transmit. (Conjecture) The adapter will initiate TX immediately.
2. Some slots of the TXFIFO queue are already filled with frames to be transmitted. (Conjecture) The adapter will store the newly written TX descriptor pointer in its internal data structure (presumbly a circular array of length 8), and start to transmit this portion of frames as soon as previously queued frames are transmitted.
### TXFIFO queue has no empty slot
This indicates the adapter has no space to memorize the start point of this new list of frames. The ath9k driver will append the new list of frames to a driver specific data structure ```txq->axq_q``` (a variable storing any unfitted, need-to-be-transmitted frames due to TXFIFO queue is full), and call function ```ath9k_hw_set_desc_link``` to chain the "last frame buffer from ```txq->axq_q```" (which is ```txq->axq_link```, this variable always stores the last frame buffer in ```txq->axq_q```) with the first frame buffer in the new list of frames.

As long as there is no empty slot in TXFIFO queue, any new list of frames will be appended and chained to the end of ```txq->axq_q```, composing a new large list of frames to be transmitted later.

Once an empty slot in TXFIFO queue is available, the list of frames in ```txq->axq_q``` will be moved to the empty slot in TXFIFO queue, and be transmitted by the adapter in the near future. This process is done at __TX finish procedure__ and I will explain in more details there.

Note: list of frames is a concept in the ath9k driver, its data structure is linked list and uses standard ```list_node``` data structure in linux kernel programming. However, this list of frames may not necessarily be chained together in the eye of the adapter, because the adapter (this part is conjecture) uses TX descriptor (aka frame buffer) to navigate between frames.  

Conjecture: the function ```ath9k_hw_set_desc_link``` is a hardware specific operation so it is close sourced, but I can infer that its functionality is to setup a link between two frame buffers so that the adapter can navigate through them.
## TX finish procedure
After the adapter finishes processing each TX descriptor (frame buffer), it will emit a TX_OK interrupt to signal the ath9k driver. The ath9k driver will then fetch status infromation from the adapter and update statistic information. The whole procedure is done in function ```ath_tx_edma_tasklet```, which will be executed each time a TX_OK interrupt is emitted.

In function ```ath_tx_edma_tasklet```, the ath9k driver will first get the list of frames from TXFIFO queue by index ```txq->txq_tailidx```, this is the fifo list that is currently being processed by the adapter. Then, the ath9k driver will "pop the first entry from the fifo list" (I am using double quote because the actual code is more complicated, but this is what essentially it does). In the end, if the fifo list becomes empty, the ath9k driver will release that slot in the TXFIFO queue. After that if the list of frames in ```txq->axq_q``` is not empty (indicating there are unfitted, need-to-be-transmitted frames due to TXFIFO queue was full), the ath9k driver will call ```ath_tx_txqaddbuf``` with ```internal``` parameter to "move ```txq->axq_q``` to that empty TXFIFO slot.

# Modified ath9k TX path
In the protocol design of UCSMA, one key requirement is to stop TX as soon as possible. Luckily, there is a FORCE_QUIET_COLLISION bit in the adapter's MAC_PCU_MISC_MODE register, and according to AR9331 datasheet, setting this bit will make "the PCU thinks that it is in quiet collision period, kills any transmit frame in progress, and prevents any new frame from starting.

In the process of my research, I need to do validation experiments to verify that transmission is indeed intercepted once the FORCE_QUIET_COLLISION bit is set. However, finding the right timing to set this bit is challenging because the actually transmission is happening inside the adapter's internal control logic and not exposed to the ath9k driver. That is to say, ath9k driver is only in charge of pushing new frames into hardware queues and telling the adatper there are new frames, but as for when exactly the adapter will transmit the newly pushed frames, it is unknown to the ath9k driver. For example, the adapter may be transmitting a previously pushed frame when you push a new frame to the hardware, so if you want to set FORCE_QUIET_COLLISION bit to intercept the transmission of the newly pushed frame, you won't know the exact timing.
## Modified Driver Code
Fortunately, I was able to modify the ath9k driver code and gain more control to the transmission procedure, which made conducting validation experiments possible.

The main conflict here is that when pushing a new frame to the hardware, a previously pushed frame may be under transmission, introducing uncertainty of when will the newly pushed frame starts being transmitted. To solve this conflict, I can just eliminate this situation, i.e. never push a new frame to the hardware unless there is no frame currently being transmitted.

In order to achieve the above feature, I first change the size of TXFIFO queue to 1, which forces the adapter to only memorize a single start point of frames. Secondly, I make sure that the list of frames moved to TXFIFO queue will only contain one frame, so that the adapter will not automaticly navigate through multiple frames on its own. Thirdly, in the TX finish procedure, instead of moving the whole ```txq->axq_q``` list to the emptyed TXFIFO slot, I only take the first frame from ```txq->axq_q``` and move it to TXFIFO queue. The patch I made for these modifications can be viewed [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/fc7aa359602ffeeca64fced72232e773d2334e9e).

At this point, I can safely proceed to my validation experiments because now I can infer the timing I need to intercept a given frame. (The following is conjecture) By the design of the adapter, once I push a new frame into the hardware for transmission, it will start to transmit immediately if environmental conditions are satisfied (i.e. no other onging transmissions or interference on the channel for AIFS period before the new frame is pushed into the hardware). For a better understanding of this part please refer to [this page](https://gist.github.com/JackWindows/c7921b76ebc6f3f960b3a2a867435637).

## Modified ath9k TX path Flowchart
![alt text](https://docs.google.com/drawings/d/1EvdWguH4Asuw1ZN8tBz769f7_8zPqGrYd1m4P7p_6u0/pub?w=960&h=720 "ATH9K Modified TX path")