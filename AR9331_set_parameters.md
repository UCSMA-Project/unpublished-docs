# Set Parameters for AR9331 Wireless Adapter
This document gives a breif introduction of how the software (operating system) controls the hardware (Wireless adapter), and give a few examples on how to set parameters for the wireless adapter.
## Basic Architecture between Software and Hardware
The core component for the software to communicate with the hardware is registers on the wireless adapter. Basically all the parameters of the wireless adapter are stored in its registers. The registers are on a block of direct accessable memory to both operating system and the wireless adapter. The OS kernel can write values to the registers and the wireless adapter can fetch values and behave accordingly.
## Examples of Setting Parameters for the Wireless Adapter
During my research, I need to manually control some parameters to accmondate my requirements in experiments. The following lists all the parameters I have manually adjusted.
### Config Contention Window/AIFS/Burst Duration
There are 10 hardware queues in the AR9331 wireless adapter, each queue has its own parameters for contention window/AIFS and burst duration. To simplify the process of setting these parameters, I tweaked an existing user space wireless tool `iw` to help accomplish this, the patch I made is available [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/a08fbfa48ac4519b2bd8fa98d9ed00fd0c353dbf). Later on I further tweaked this tool and enable it to set contention window for wireless adapters in pure monitor mode, the patch is [here](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/d3c375feb36a0c5cac86de794e15c931fb3cbf4e).  
The basic usage for setting contention window is as follows:  
```
root@board3:~# iw mon0 set wmm_params
Usage:  iw [options] dev <devname> set wmm_params <VO|VI|BE|BK> <cw_min> <cw_max> <aifs> <txop>

Set WMM parameters for a queue (VO, VI, BE, or BK):
 * cw_min/cw_max - contention window min/max (slots)
 * aifs - interframe spacing (us)
 * txop - time for tx operation limit (in units of 32 microseconds), 0 = off
```
### Config Noise Floor
The noise floor is not just a single value in a register. To config noise floor, you need first turn off the automatic calibration process in the ath9k driver, then write the value into certian register in the wireless adapter, at last you set some control bits in other registers to tell the wireless adapter to load the previously written noise floor. To ease the whole process, I backported a [patch](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/791d942f048f0291eea2f0c0f9f45e3f86d11edd) from modern ath9k project to accomplish this task.  
To override the automatically calibrated noise floor, use the following command:  
`echo [noise floor, e.g. -70] > /sys/kernel/debug/ieee80211/phy0/ath9k/nf_override`
### Read/Write Registers
Sometimes you may encounter a situation where you just want to read/write a single register. The ath9k driver already provided an interface of doing this. You can use  
`echo [the address of register] > /sys/kernel/debug/ieee80211/phy0/ath9k/regidx`  
and then use  
`cat /sys/kernel/debug/ieee80211/phy0/ath9k/regval`  
to read register value or use  
`echo [register value] > /sys/kernel/debug/ieee80211/phy0/ath9k/regval`  
to write register value.

To simplify the process, I also developed two [shell scripts](https://github.com/JackWindows/OpenWRT-14.07-JS9331/tree/master/helper_scripts) `rget` and `rset`. You can put these two scripts into `/usr/bin/` folder of the AR9331 development board and use them to read/write registers more conveniently. The usage for these two scripts are as follows:  
```
root@board3:~# rset
usage: /usr/bin/rset <reg index> <value>
root@board3:~# rget
usage: /usr/bin/rget <reg index>
```
### Config Max CCA Power
Maximum CCA Power is crucial when creating the [desired topology](https://gist.github.com/JackWindows/1bb9073d277f301bdec711e1f91c4525#setup-desired-topology). Therefore, I added some parsing functions in my `rget` and `rset` scripts. To read current MaxCCAPower, use command `rget 0x9e1c`. To write MaxCCAPower value, use command `rset 0x9e1c maxCCA [value, e.g. -100]`.

## Initialize Wireless Adapter
To best accommodate the requirements in my test environment, I wrote a [initialization script](https://github.com/JackWindows/OpenWRT-14.07-JS9331/blob/master/helper_scripts/setup_mon0) for wireless interface, which is put into the `/etc/hotplug.d/net/` on AR9331 development board.

The wireless adapter is put in pure monitor mode with interface name `mon0`. Once interface `mon0` comes online, the `setup_mon0` script will be automatically executed and set a bunch of parameters for the wireless adapter. For more details on what parameters have been set, checkout the comments in script.