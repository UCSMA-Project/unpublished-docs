These tests are aimed at testing the hardware's behavior under different parameters. Basic parameters are set as below:
```
frame length: 1000 bytes
frame duration: 920us (192us preamble + 728us data)
SIFS: 8us (10us - 2us)
Slot Time: 20us (long slot)
```
# Single Node Experiments
## CW: 0 AIFS: 0
### Observed Rate: 1040pps
### Theoretic Rate: 1077pps
```
average processing time per frame: 920us(frame duration) + 8us(SIFS)
theoretic rate: 10000000us / 928us ~= 1077pps
```
In frame transmission timeline view, I noticed there is always a 30~50us delay between frames. The driver takes \<10us between receiving TX_OK interrupt from previous frame and putting next frame into hardware TX buffer, and from another observation I know typically the TX_OK interrupt arrives around 30us later after the frame has actually been transmisitted. Combined, these two factors can add ~40us processing time to each frame, and this coincides to the observed rate of 1040pps, which is equivalent to a processing time of 961.5us per frame, indicating the delay between frames is ```961.5 - 920 = 41.5us```.
### Negative Effect of Modified ATH9K TX path
In my modified driver, TX path is slightly changed to help monitoring TX status. The modification is to put only 1 frame in hardware TX buffer each time, and only put next frame if the driver receives TX_OK interrupt for the previous frame.

However, this change introduces additional delays of interrupt handling and putting next frame to buffer, which add ~40us delay between frames. In the original driver, this issue doesn't occur because multiple frames are put into hardware TX buffer at the same time. When the adapter finishes transmitting a frame, it can instantly proceed to next frame without waitting the drvier to put next frame into hardware TX buffer.

In the same experiment conducted on an original TX path driver, I was able to observe a rate of __1075pps__, which is closer to the theoretic result.

Note that this performance degradation only affects some extreme situations (i.e. the IFS, inter frame spacing, is less than the introduced delay). For regular 802.11 BestEffort queue with AIFS equals to 3 (equivalent of 68us if long slottime is used), the adapter has to wait at least 68us before next transmission due to post-frame backoff, in that case the introduced processing delay won't be a problem at all (as long as the driver can put next frame into hardware TX buffer within 68us).

## CW: 15 AIFS: 3
This is the default parameters for 802.11 BestEffort queue.
### Observed Rate: 866~877pps
### Theoretic Rate: 878.7pps
```
average processing time per frame: 920us(frame duration) + 68us(AIFS) + 150us(avg backoff) = 1138us
theoretic rate: 10000000us / 1138us ~= 878.7pps
```
Under this parameter setup, the AIFS(68us) is well beyond the processing delay(30~50us) I introduced in the driver. Therefore, there is nearly no performance loss due to single TX buffer in the driver implementation. My experiments on original and modified TX path also confirm this, with or without the TX path modification, the observed rate is constantly 877pps.

# A Special Note for SIFS
During examing driver code, I find an interesting behavior of handling SIFS (Shortest Inter Frame Spacing) register: the actual SIFS value written into the hardware register is actually 2us less than the stored value in driver (i.e. when the SIFS value is ```10us``` in the drvier, the actual value written to the register is ```10 - 2 = 8us```).

The idea behind this special handling is yet to be explained, but from the results of the above two experiments, I may have an explanation. I have notcied a slight inconsistency between observed rate and theoretical rate (e.g. theoretical ```1077pps``` v.s. observed ```1075pps``` for ```CW = 0, AIFS = 0```, and theoretical ```878.7pps``` v.s. observed ```877pps``` for ```CW = 15, AIFS = 3```). Note that in the calculation of theoretical rate, I used the actual SIFS value written to the register (```8us```) instead of the SIFS value stored in the driver (```10us```). If I use ```10us``` as SIFS value in calculation, then the theoretical rate would be ```1075.3pps``` and ```877.2pps```, which coincides with the observed rate, indicating the special handling of SIFS register may be a mechanism to compensate the lost performance due to hardware implementation.

# Double Nodes Experiments
This set of experiments is mainly used to validate some results I expect, as a precondition check before proceeding to triple nodes experiments.

Both nodes use the same TX queue parameters of ```CW = 15, AIFS = 3```.
## Edge and Mid Nodes
In this experiment I let the mid node and one edge node transmit as fast as possible, so they will compete for the channel, and observe the transmission rate to see if it matches what I expect.
### Expected Behavior
The two nodes will share access to the channel on a 50% v.s. 50% basis, resulting the same transmission rate on both nodes.
### Observed Rate
Edge Node: ```494~502pps```  
Mid Node: ```493~503pps```  
Rates are based on per 100 seconds statistics, the observed rate fluctuates a little bit during experiment.
### Simulated Rate
Edge Node: ```496~498pps```  
Mid Node: ```496~498pps```  
Simulations use exactly same parameters as real experiments and assume nodes can sense each other perfectly (i.e. two nodes will only start transmission simultaneously iff they start at exactly the same time in us). Simulation duration is 100 seconds, rates are averaged over the whole duration.
### Insight gained on result
Generally speaking, the observation matches expected behavior pretty well, both nodes achieved similar transmission rate, within reasonable fluctuation.

However, the combined transmission rate of both nodes is great than the transmission rate when they transmit indvidually, which seems a little bit strange. But on a second thought this behavoir seems to be legit, when two nodes are both competing for the channel, there is a possibility that they generate the same backoff value and start to transmit simultaneously, which contributes to both nodes' transmission rate, leading to a higher overall transmission rate.

In this experiment, the contention window is 15, and backoff value is generated uniformly between 0 and 15, which indicates the possbility of winning (generated backoff value is smaller or equal to the other node's) is
```
P = \sum_{n=0}^{15} 1/16 * (n + 1)/16 ~= 59.77%
```
So during the 877 competing periods in 1 second, there would be ```877 * 59.77% ~= 524``` periods that a node wins. Note this is just a rough estimation since I assume both nodes will generate a new backoff value in each competing period, but in protocol implementation backoff value is generated per frame (i.e if a node failed in a competing period, it will re-use the remaining backoff value in next competing period), which makes the odds different from what I calculated here.
## Edge and Edge Nodes
In this experiment I let both edge nodes transmit as fast as possible, and see it the outcome matches expectation.
### Expected Behavior
Both nodes will achieve their maximum transmission rate as in the Single Node Experiment, there should be zero interference between edge nodes.
### Observed Rate
Edge Node1: ```875~878pps```  
Edge Node2: ```875~878pps```  
Rates are based on per 100 seconds statistics, the observed rate fluctuates a little bit during experiment.
# Triple Nodes Experiment
In this experiment I let all three nodes transmit as fast as possible, see see how the channel access is divided between the mid node and two edge nodes.  
All three nodes use the same TX queue parameters of ```CW = 15, AIFS = 3```.
## Expected Behavior
This desired behavior is very hard to predict right now, but with a simulation tool I worte, the expected result should be:  
Mid Node: ```86~88pps```  
Edge Nodes: ```804~805pps```
## Observed Rate
Mid Node: ```62~77pps```  
Edge Nodes: ```815~831pps```
# Triple Nodes Experiment with RX disabled
In this experiment the RX on all wireless chips is disabled. This will let the wireless chips focus on TX and not get blocked by RX process, it also implies disabling NAV (Network Allocation Vector) as well as disabling all RX related actions (e.g. EIFS).  
All three nodes use the same TX queue parameters of ```CW = 15, AIFS = 3```.
## Observed Rate
Mid Node: ```165~172pps```
Edge Nodes: ```742~748pps```
Rates are based on per 100 seconds statistics, the observed rate fluctuates during experiment.