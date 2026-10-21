# Hackintosh on a Dell Optiplex 9020 Micro
Desktop Hackintosh build based on [Dell Optiplex 9020 Micro](https://dl.dell.com/topicspdf/optiplex-9020m-desktop_Owners-Manual_en-us.pdf) running macOS 15 Sequoia.

## Project in a Summary

### Background

Bought this compact size desktop during COVID and had my first Hackintosh experience by installing macOS 10.15 Catalina.  Upgraded to macOS 12 Monterey in late 2022 and running smooth and stable.  However, OS security and third party applications updates are limited in 2026, and hence upgrading to a newer macOS becomes necessary.  

### Key H/W Spec

- Intel Core i5-4590S (Haswell architecture, 4C4T, 65W)
- Intel HD Graphics 4600
- Apple Inc. BCM94360CS2 on a proprietary Apple AirPort 6+12 pins connector
- 2x PACCOM 8GB DDR3L module (SK Hynix H5TC4G83BFR-PBA memory chip) 
- Sandisk X400 SATA3 SSD

### Working, Not Working, Not Tested, and Issues

**Working**
- CPU Speed Step, Turbo Boost, and all core and package C-states residency
- iGPU with Metal 2 acceleration
- WiFi and Bluetooth by the native Apple/Broadcom WiFi/BT combo card supporting AirDrop, Continuity and Handoff
- SSD Trim and SMART reporting
- Ethernet at 1Gbps
- Audio in/out from internal speakers, headphone jack and microphone as well as HDMI Audio
- All USB ports supporting USB2.0 (480Mbps) and USB3.0 (5Gbps)
- Sleep and wake without dropping WiFi or Bluetooth
- iCloud sync services like storage, photos, notes, passwords etc.

**Not Working**
- Full [System Integrity Protection](https://support.apple.com/en-sg/102149) ("SIP")
- Full [Apple Mobile File Integrity](https://theapplewiki.com/wiki/AppleMobileFileIntegrity) ("AMFI")
- [Apple Secure Boot](https://dortania.github.io/OpenCore-Install-Guide/config.plist/security.html#misc)
- DRM contents
- SideCar (since HD Graphics 4600 does not support HEVC/H.265)

**Not Tested**
- iMessages (well not everyone falls into Apple ecosystem)
- FaceTime / Photobooth (since I don't have a webcam)

**Issues**

- Occassionally the screen goes dark for 1-2 seconds and then recover by itself


## The Preparation

**Ventura, Sonoma, Sequoia or Tahoe...?**

Brief hardware compatibility study against newer x86_64 compatible macOS:

|Hardware|Monterey|Ventura|Sonoma|Sequoia|Tahoe|
|:---|:---|:---|:---|:---|:---|
|CPU: i5-4590S (AVX2)|Not required|Required|Required|Required|Required|
|GPU: HD Graphics 4600|Support|via RP|via RP|via RP|Under development|
|WiFi/BT: BCM94360CS2|Native|Native|via RP|via RP|via RP|

RP = Root patching

Apple dropped Haswell iGPU display driver since Ventura and Broadcom PCIe WiFi stack since Sonoma respectively, and hence [OpenCore Legacy Patcher](https://github.com/dortania/Opencore-Legacy-Patcher) root patching is inevitable.  Fortunately Intel introduced AVX2 to Haswell architecture so the aging i5-4590S is not that dated.  Since anything beyond Monterey will require root patching, going all the way to Sequoia (before OCLP 3.0 STABLE release) helps address my primary goal before retiring this box.

**IMPORTANT NOTE:**  OCLP root patching requires lowering "SIP", breaking "AMFI" and disabling Apple Secure Boot.  Please understand the implications and risks before proceeding.

**Which SMBIOS**

|SMBIOS|CPU TDP|Graphics|Display|Last Supported OS|
|:---|:---|:---|:---|:---|
|Macmini7,1|i5-4278U @ 28W or i5-4260U @ 15W|Intel Iris Graphics 5100 (0x0A2E) or Intel HD Graphics 5000 (0x0A26)|External|macOS 12 Monterey|
|iMac14,1|i5-4790R @ 65W|Intel Iris Pro Graphics 5200 (0x0D22)|Internal|macOS 10.15 Catalina|
|iMac14,4|i5-4260U @ 15W|Intel HD Graphics 5000 (0x0A26)|Internal|macOS 10.15 Catalina|

During the Monterey installation, SMBIOS Macmini7,1 was the natural choice because the box 1) is on Haswell architecture; 2) runs with a 35W TDP cap which is quite close to 28W of i5-4278U, hence similar power and thermal management; 3) has no dGPU; and 4) most importantly, runs Monterey natively without SMBIOS spoofing or additional kexts/boot-args.

To upgrade to Macmini7,1 unsupported Sequoia, there are two paths - 1) spoof to a SMBIOS that allows native Sequoia installation or 2) stick to Macmini7,1 by using a combination of RestrictEvents.kext with ```revpatch=sbvmm``` as well as ```-no_compat_check``` in boot-args.  Macmini7,1 frequency vector has been giving me stable power and thermal performance, and hence option 2 makes more sense.

P.S. I am just too lazy tuning frequency vector by CPUFriend

## The Process

### BIOS Settings and Mods

My box came with BIOS version A19.  During Monterey installation, it had been flashed with [version A25](https://www.dell.com/support/home/en-gb/drivers/driversdetails?driverid=cf8tx), which is still latest one.  Thanks to [hoaug-tran's](https://github.com/hoaug-tran) repo on his [Optiplex-9020M](https://github.com/hoaug-tran/Optiplex-9020M) Hackintosh.  The [BIOS Settings](https://github.com/hoaug-tran/Optiplex-9020M#bios-settings) provides a great guide to follow.

I am also encouraged by his guide to dive into the path of BIOS mod.  Though BIOS modification is not strictly neccessary there are couple of benefits: 1) using less quirks in OpenCore config.plist; 2) bypassing or removing devices configuration workaround; and 3) making the system "Mac-like".

Here are the mods I made:-

1) **Disable CFG-Lock** - Allow Mac kernel to directly access CPU MSR 0xE2 register without turning on "AppleXcpmCfgLock" quirk in OpenCore config.plist. For details, please refer to [OpenCore Post-Install > Fixing CFG Lock](https://dortania.github.io/OpenCore-Post-Install/misc/msr-lock.html) guide.

2) **Increase "DVMT Pre-alloc" to 64MB** - Allow enough DVMT Pre-allocated memory for the selected framebuffer (52MB for Intel HD Graphics 4600), and hence dropping [VRAM patching]((https://dortania.github.io/OpenCore-Post-Install/gpu-patching/intel-patching/vram.html)) via Device Properties.  

3) **Running USB with xHCI only** - Turn off the hybrid EHCI/xHCI behaviour on the Haswell chipset that is required for Windows 7 and switch to a sole xHCI setup like a real Mac of the same era, hence making USB port mapping less complicated, stabilizing sleep/wake behaviour and improving core/package c-state residency by mitigating interrupt storms.

|Mod|VarStore|VarOffset|Default|Change to|Purpose|
|:---|:---|:---|:---:|:---:|:---|
|1|0x2|0xD9F|0x1|0x0|Disable CFG-Lock (MSR 0xE2)|
|2|0x2|0x263|0x1|0x2|Change DVMT Pre-alloc to 64MB from 32MB|
|3a|0x2|0x144|0x3|0x1|Change XHCI mode to "Enabled" from "Smart Auto"|
|3b|0x2|0x15A|0x1|0x2|Route all (USB) pins to XHCI|
|3c|0x2|0x169|0x1|0x2|Enable all USB 3.0 pins|
|3d|0x2|0x146|0x1|0x0|Disable EHCI#1 controller|
|3e|0x2|0x147|0x1|0x0|Disable EHCI#2 controller|

Syntax for setup_var_cv is ```setup_var_cv nameOfVarStore offsetInVarStore [optional variable size] [optional value to write]```.  For exmaple, to change XHCI mode to "Enabled" from "Smart Auto", the command in modGrubShell is ```setup_var_cv Setup 0x144 0x1 0x1``` .

Note: XHCI hand-off is enabled by default.  Good to check by running ```setup_var_cv USBSupport 0x1A``` and see if it returns value ```0x1```.

### OpenCore Configuration

[OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/config.plist/haswell.html) provides clear guidance.  Hands down.  Below summarizes how the EFI is configured.

<ins>SSDTs</ins>

SSDT can be generated by the great tool [SSDTTime](https://github.com/corpnewt/SSDTTime) by [corpnewt](https://github.com/corpnewt) or downloaded from [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/), one of the Dortania Guides.

|SSDT|Purpose|
|:---|:---|
|SSDT-EC|Fixes the embedded controller|
|SSDT-USBX|Specifies power properties and budget to xHCI controller to support USB3.0 devices|
|SSDT-GPRW|Intercepts GPE wake events from USB/HDEF/HDAU/LAN to improve sleep/wake behaviour; require ACPI > Patch rename to be effective|
|SSDT-SBUS-MCHC|Creates a hook for AppleSMBusController.kext|

Beacause of my Catalina and Monterey experience I used to have SSDT-PLUG in my EFI.  It is indeed ***not required*** since macOS Monterey 12.3+.

<ins>kexts</ins>

For any Hackinstosh installation by a full installer image, Lilu, VirtualSMC and WhateverGreen are a **must**.  For network install (preferably over wired connection), please put in kext enabling the NIC.  These are the minimum to get us through the installation and to the Welcome screen.  The rest can be injected post macOS installation.

|kexts|Purpose|
|:---|:---|
|[Lilu](https://github.com/acidanthera/Lilu)|Master of all kexts.  Period.|
|[VirtualSMC, SMCProcessor and SMCDellSensors](https://github.com/acidanthera/VirtualSMC)|Emulates Apple SMC and provides sensors readings plus fan control via its plug-ins|
|[Whatevergreen](https://github.com/acidanthera/WhateverGreen)|Injects framebuffer information defined in Device Properties and patches connectors information|
|[IntelMausi](https://github.com/acidanthera/IntelMausi)|Enables onboard i217-LM NIC|
|[AppleALC](https://github.com/acidanthera/AppleALC)|Provides codec support to the onboard Realtek ALC 3234|
|[USBInjectAll](https://github.com/rehabman/os-x-usb-inject-all)|Injects ***all*** USB ports under Intel EHCI/xHCI chipset using in conjunction with ```XhciPortLimit=True``` quirk; should be disabled after USB port mapping is properly done|
|USBMap|Defines USB port type; Generated by another great tool [USBMap](https://github.com/corpnewt/USBMap) by corpnewt|
|[RestrictEvents](https://github.com/acidanthera/RestrictEvents)|Enables OTA updates for unsupported macOS with boot-args ```revpatch=sbvmm```|
|[AMFIPass](https://github.com/bluppus20/AMFIPass)|Provides application integrity and virtualization security under SIP-lowered
|[IOSkywalkFamily, IO80211FamilyLegacy, and AirPortBrcmNIC](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Wifi)|Kexts required, together with enabling "Kernel > Block > Allow IOSkywalk Downgrade" in config.plist, to bring back Broadcom WiFi stack via OCLP root patching; well documented by [OpenCore Visual Beginners Guide](https://chriswayg.gitbook.io/opencore-visual-beginners-guide/advanced-topics/open-core-legacy-patcher-oclp/broadcom-wifi-on-macos-sequoia)|

<ins>Device Properties</ins>

Always use [Hackintool](https://github.com/benbaker76/Hackintool) PCIe tab to locate the correct PCI device path.

1. Enabling iGPU Metal acceleration, patching the port type for the HDMI connector board (which helps EDID handshake) and enabling HDMI audio
    - Display Controller at ```PciRoot(0x0)/Pci(0x2,0x0)```
        - ```AAPL,ig-platform-id``` = ```0300220D``` (Type: ```Data```)
        - ```device-id``` = ```12040000``` (Type: ```Data```)
        - ```framebuffer-patch-enable``` = ```01000000``` (Type: ```Data```)
        - ```framebuffer-con2-enable``` = ```01000000``` (Type: ```Data```)
        - ```framebuffer-con2-type``` = ```00080000``` (Type: ```Data```)
        - ```hda-gfx``` = ```onboard-1``` (Type: ```String```)

2.  Specifying layout-id for Realtek codec
    - Audio Device at ```PciRoot(0x0)/Pci(0x1B,0x0)```
        - ```layout-id``` = ```0d000000``` (Type: ```Data```)

3.  Ensuring AppleLPC.kext is loaded for power management optimization by spoofying LPC controller device-id to Z87 Express LPC Controller ```pci8086,8c44``` from unsupported onboard Q87 Express LPC Controller ```pci8086,8c4e``` 
    - ISA Bridge (LPC Controller) at ```PciRoot(0x0)/Pci(0x1F,0x0)``` 
        - ```device-id``` = ```448C0000``` (Type: ```Data```)
        - ```compatible``` = ```pci8086,8c44``` (Type: ```String```)

4.  ASPM is enabled by default in Monterey but not in Sequoia (and I am not sure why), and hence the following is inserted for my box:
    - PCI Bridge (PCI Express Root Port #1) at ```PciRoot(0x0)/Pci(0x1C,0x0)```
        - ```pci-aspm-default``` = ```03000000``` (Type: ```Data```)
    - PCI Bridge (PCI Express Root Port #4) at ```PciRoot(0x0)/Pci(0x1C,0x3)```
        - ```pci-aspm-default``` = ```03000000``` (Type: ```Data```)
    - Network Controller (BCM4360 Wireless Adapter) at ```PciRoot(0x0)/Pci(0x1C,0x3)/Pci(0x0,0x0)```
        - ```pci-aspm-default``` = ```03000000``` (Type: ```Data```)

<ins>Quirks</ins>

The EFI is configured mostly in line with OpenCore Install Guide, except for below:-

- Kernel
    - ```AppleXcpmCfgLock``` = ```False``` if CFG-Lock is disabled by BIOS mod
    - ```CustomSMBIOSGuid``` = ```False``` 
    - ```DisableIoMapper``` = ```False``` if Intel VT-d is disabled in BIOS settings
    - ```XhciPortLimit``` = ```True``` when working on USB port mapping **only**
    - ```ThirdPartyDrives``` = ```True``` is required to enable TRIM on my Sandisk X400 SATA3 SSD

<ins>boot-args</ins>

During installation and root patching, use ```-v keepsyms=1 debug=0x100 -no_compat_check``` in boot-args.  Change to ```keepsyms=1 -no_compat_check``` once the system is considered stable

<ins>OCLP root patching</ins>

csr-active-config is defaulted to ```00000000``` (meaning SIP fully enabled).  For persisting root patch, change to ```03080000```, which is regarded as "SIP-lowered".



---

<details>

<summary>Hardware</summary>

<ins>Motherboard</ins>
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

<ins>Components installed</ins>
||Device|Notes|
|:---|:---|:---|
|CPU and iGPU|Intel Core i5-4590S with HD Graphics 4600|Replaced by seller|
|Memory|A-Data DDR3L 1,600Mhz 8GB x2||
|WiFi/BT|Apple Inc. BCM94360CS2|via a m.2 A+E key to Apple Airport adapter|
|Storage|Sandisk X400 256GB SSD|using onboard m.2 B key slot|
|Flexiport|Dell RH40T HDMI I/O port|Bye bye DP-to-HDMI passive adapter|

Note: Onboard VRM, stock power brick and BIOS/firmware are designed to support 35W TDP T-series CPU (e.g. i5-4590T, i7-4785T).  The current 65W TDP i5-4590S is not sufficiently powered.
</details>

<details>

<summary>Tools for BIOS mod</summary>

Here are the tools for BIOS mod

- [BIOSUtilities](https://github.com/platomav/biosutilities) - Collection of various BIOS/UEFI-related utilities which aid in research and/or modding purposes

- [UEFITool](https://github.com/longsoft/uefitool) - UEFI firmware image viewer and editor

- [-IFRExtractor](https://github.com/LongSoft/IFRExtractor-RS) - Rust utility to extract UEFI IFR data into human-readable text

- [modGrubShell](https://github.com/datasone/grub-mod-setup_var) - A modified grub allowing tweaking hidden BIOS settings

</details>
