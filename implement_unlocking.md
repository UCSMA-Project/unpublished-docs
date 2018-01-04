In this document I will describle the unlocking mechanism of UCSMA and explain how to implement it in the ath9k driver.
# Unlocking Kernel Module
My implementation of unlocking mechanism is in the form of a LKM (loadable kernel module), the source code is [here](https://github.com/JackWindows/ucsma-kernel_module/blob/master/unlock.c) and [this](https://gist.github.com/JackWindows/8ab85dd00bcef08242d818fce440d94f#steps-to-compile-the-unlocking-kernel-module) is the procedures to compile it. This kernel module is depend on the ath9k drvier and requires some modifications of driver code in order to work:
1. Export the ath9k hardware abstract data structure ([this patch](https://github.com/JackWindows/OpenWRT-14.07-JS9331/commit/76def186a1ffde560ae580fd9771395bc61ff7d0)). This will expose the hardware abstract structure to other kernel modules in the OS kernel and enable the unlocking kernel module to read and write hardware registers of the wireless chip.
2. Add data structures for logging. This will allow me to monitor the timing for TX (e.g. when a frame is pushed into hardware buffer, when a TX_OK interrupt is triggered, when an unlocking is performed) and output logs to the dmesg of the OS kerenl.

Note that there is a special treatment for the timer in the unlocking kernel module. Normally a kernel module uses a regular timer defined in the linux kernel header file ```linux/timer.h```, however, this timer only has precision in the ```jiffy``` level, which is 10ms on a kernel with clockrate 100Hz. In the unlocking implementation it requires a timer with precision in the level of microsecond. After some research I found that in newer linux kernels there is already a component called ```hrtimer``` ([high-resolution timer](https://www.kernel.org/doc/Documentation/timers/hrtimers.txt)) implemented, therefore, it is important to use hrtimer in the unlocking implementation to achieve a precise control of the timing of unlocking.
## Unlocking System Architecture
![alt-text](https://docs.google.com/drawings/d/1SeXv05ToQIvNfj1iuyf3ThWUeTOXHY7oCDSkLs1gNI0/pub?w=1018&h=702 "Unlocking System Architecture")  
The unlocking kernel module will directly acess GPIO hardware and indirectly access wireless adapter through ath9k driver. The unlocking control signal is sent via GPIO wires, and the unlocking of wireless adapter is performed by setting FORCE_QUIET_COLLISION register.
## Unlocking Flowchart
![alt-text](https://docs.google.com/drawings/d/1UI78SshYlR1KA02dbUqrSiKcIhYAM2THyE-qz87QMKs/pub?w=1139&h=1467 "Unlocking Flowchart")  
There are three components in the flowchart:
1. __Load kernel module__. When initiate the kernel module, it will create a timer and set its expiration, then register a GPIO interrupt handler.
2. __Timer invocation handler__. When timer is invoked, the kernel will send an unlocking signal through GPIO, which will in turn trigger the interrupt handler.
3. __GPIO IRQ handler__. At the end of the GPIO IRQ handler, timer is re-activated, which will lead to another unlocking signal in the future.
## Unlocking Illustration
![alt-text](https://docs.google.com/drawings/d/1m-MRZbyyS-NzPD9BThxAAaBWn1jk_4sYJu-3QC-MzOA/pub?w=1305&h=990 "Unlocking Illustration")  
1. Unlocking signal is transmitted via wired GPIO wires.
2. Each node in the network needs to relay the unlocking signal it received, this ensures unlocking signal will propgate throughout the whole network.
3. GPIO interrupt handler can be triggered by either a timer expiring event or a relayed GPIO unlocking signal.

# Appendix
## Steps to compile the unlocking kernel module
The unlocking kernel module is in the form of a loadable kernel module, to compile it requires the following setps:
1. First thing is to build and install the OpenWrt cross compile toolchain, which will prepare the necessary linux header files to compile a loadable kernel module in the correct CPU architecture. There is an [article](https://wiki.openwrt.org/doc/devel/crosscompile) on OpenWrt official website about how to do this.
2. Second thing is to gather the underlying ath9k drvier code. Note that OpenWrt 14.04 has various patches done to the original ```compat-wireless-2014-05-22``` driver code, so it is important to obtain the __already patched__ ath9k driver code instead of the original one. I did this by executing ```make package/mac80211/{clean,prepare}``` in the OpenWrt buildroot directory, which will extract the ```compat-wireless-2014-05-22``` driver code and apply all existing patches to it.
3. After step 2 the ath9k drvier code should be in folder  
    ```
    build_dir/target-mips_34kc_uClibc-0.9.33.2/linux-ar71xx_generic/compat-wireless-2014-05-22/drivers/net/wireless/ath/ath9k
    ```  
    Copy all the files in the folder to a new folder, and also put the source code of the unlocking kernel module in the new folder.
4. Modify the Makefile to tell the compiler to compile a loadalbe kernel module. The Makefile I use is [here](https://github.com/JackWindows/ucsma-kernel_module/blob/master/Makefile), the highlighted part is to add ```obj-m += unlock.o``` (which tells the compiler to look for ```unlock.c``` and compile it to a LKM) and the following code to the end of Makefile:
    ```
    all:
	    make -C "~/OpenWRT-14.07-JS9331/build_dir/target-mips_34kc_uClibc-0.9.33.2/linux-ar71xx_generic/linux-3.10.49/" \
            ARCH=mips \
            CROSS_COMPILE="/opt/OpenWrt-Toolchain-ar71xx-for-mips_34kc-gcc-4.8-linaro_uClibc-0.9.33.2/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2/bin//mips-openwrt-linux-" \
            M=/home/kernel_module/ath/ath9k     \
            modules
    ```
    This will create a new compile target and tell the compiler where to look for the cross compile prefix.
5. Compile the loadable kernel module by executing ```make``` in the folder. This will generate ```unlock.ko``` and that is the executable binary of the unlocking LKM. Now transfer the file to the development boards and install the kernel module by executing ```insmod unlock.ko```.