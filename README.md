# Hackintosh on a Dell Optiplex 9020 Micro
Desktop Hackintosh build based on [Dell Optiplex 9020 Micro](https://dl.dell.com/topicspdf/optiplex-9020m-desktop_Owners-Manual_en-us.pdf)

<ins>On-board Features</ins>

||Device|Notes|
|:---|:---|:---|
|Chipset|Intel Q87 Express|Haswell architecture with enterprise feature e.g. vPro|
|CPU Socket|Intel LGA1150|Support Intel 4th Generation Haswell architecture|
|Memory|Dual SO-DIMM sockets|Support 2GB, 4GB and 8GB Non-ECC DDR3L RAM; Max at 16GB|
|Sound|Realtek HDA ALC3234||
|Audio|Internal speaker, mic and headset||
|Ethernet|Intel i217-LM|RJ45 10/100/1000bps NIC|
|USB|6x USB3.0 and 1x USB2.0|3.0: 2 front & 4 rear Type A ports; 2.0: via m.2 A+E key slot| 
|WiFi/BT|m.2 A+E key 2232|PCIe 2.0 x1 + USB 2.0|
|Storage|m.2 B key 2280|Support SATA but not NVMe SDD|
|Storage|2.5-inch drive bay|Support SATA HDD/SDD|
|Flexiport|Dell proprietary|Support optional HDMI, DP, Serial or PS/2 connector boards|

<ins>Components/Parts installed</ins>

||Device|Notes|
|:---|:---|:---|
|CPU and iGPU|Intel Core i5-4590S with HD Graphics 4600|Replaced by seller|
|Memory|A-Data DDR3L 1,600Mhz 8GB x2||
|WiFi/BT|Apple Inc. BCM94360CS2|via a m.2 A+E Key to Apple proprietary adapter|
|Storage|Sandisk X400 256GB SSD|using onboard m.2 B key slot|

Note: Onboard VRM, stock power brick and BIOS/firmware are designed to support 35W TDP T-series CPU (e.g. i5-4590T, i7-4785T).  The current 65W TDP i5-4590S is not sufficiently powered.

-----
**Background**

<ins>Hardware Compatibility</ins>

Had been running macOS 12 Monterey smooth and stable since 2022 however OS security and third party applications updates and support are limited in 2026, and hence upgrading to a newer macOS becomes necessary.  Hardware compatibility was studied against various newer x86_64 compatible macOS:

|Hardware|Monterey|Ventura|Sonoma|Sequoia|Tahoe|
|:---|:---|:---|:---|:---|:---|
|CPU: i5-4590S (AVX2)|Not required|Required|Required|Required|Required|
|GPU: HD Graphics 4600|Support|RP required|RP required|RP required|RP under development|
|WiFi/BT: BCM94360CS2|Native|Native|RP required|RP required|RP required|

RP = Root patching

Apple dropped Haswell iGPU display driver since Ventura and Broadcom PCIe WiFi stack since Sonoma, and hence [OCLP](https://github.com/dortania/Opencore-Legacy-Patcher) root patching is inevitable with the above setup.  Fortunately Intel introduced AVX2 to Haswell architecture so the aging i5-4590S is not that dated.  Since anything beyond Monterey will require root patching, going all the way to Sequoia (at least for now) helps address my primary goal before retiring the box.

<ins>SMBIOS</ins>

|SMBIOS|CPU TDP|Graphics|Display|Last Supported OS|
|:---|:---|:---|:---|:---|
|Macmini7,1|i5-4278U @ 28W|Intel Iris Graphics 5100 (0x0A2E)|External|macOS 12 Monterey|
|iMac14,1|i5-4790R @ 65W|Intel Iris Pro Graphics 5200 (0x0D22)|Internal|macOS 10.15 Catalina|
|iMac14,4|i5-4278U @ 28W|Intel Iris Graphics 5100 (0x0A2E)|Internal|macOS 10.15 Catalina|

During the early Monterey installation, SMBIOS Macmini7,1 was the natural choice because the box 1) is on Haswell architecture; 2) runs with a 35W TDP cap which is quite close to 28W of i5-4278U, hence similar power and thermal management; 3) has no dGPU; and 4) most importantly, runs Monterey natively without SMBIOS spoofing or additional kexts/boot-args.

To upgrade to unsupported Sequoia, there are two paths - 1) spoof to a newer SMBIOS or 2) stick to Macmini7,1 by using a combination of [RestrictEvents](https://github.com/acidanthera/RestrictEvents) and revpatch=sbvmm as well as boot-args -no_compat_check.  To enjoy the same power and thermal management option 2 makes more sense.

-----

**Configuration**

<ins>BIOS Settings</ins>

Thanks to [hoaug-tran's](https://github.com/hoaug-tran) repo on a similar [Hackintosh setup](https://github.com/hoaug-tran/Optiplex-9020M), I dived into the path of BIOS modding by using tools like [UEFITool](https://github.com/longsoft/uefitool), [IFRExtractor](https://github.com/LongSoft/IFRExtractor-RS) and [modGrubShell](https://github.com/datasone/grub-mod-setup_var).

Below are the changes I made:-

|VarStore|VarOffset|Default|Recommended|Reason|
|:---|:---|:---:|:---:|:---|
|0x2|0xD9F|0x1|0x0|Disable CFG Lock to enable direct CPU power management|
|0x2|0x263|0x1|0x2|Change DVMT Pre-alloc to 64MB from 32MB|
|0x2|0x144|0x3|0x1|Change XHCI mode to Enabled from Smart Auto|
|0x2|0x15A|0x1|0x2|Route all (USB) pins to XHCI|
|0x2|0x169|0x1|0x2|Enable all USB 3.0 pins|
|0x2|0x146|0x1|0x0|Disable EHCI#1 controller|
|0x2|0x147|0x1|0x0|Disable EHCI#2 controller|
