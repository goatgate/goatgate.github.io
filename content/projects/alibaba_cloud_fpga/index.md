--- 
title: "Alibaba cloud FPGA: the bargain bin UltraScale+"
description: "Using a decommissioned Alibaba cloud accelerator card as an FPGA dev board"
summary: "No documentation, no problem!"
tags: ["fpga", "ebay", "debugging", "linux", "hacking"]
date: 2025-09-02
draft: true
showTableOfContentse: true
---

# Introduction

I was recently in the market for a new FPGA to start building my upcoming projects on. 
 

Due to the scale of my upcomming projects a Xilinx series 7 UltraScale+ FPGA of the Virtex family would be perfect,
 but a Kintex series FPGA will be a sufficent for early prototyping.
Due to not wanting to part ways with the eye watering amounts of money that is
required for an Vivado enterprise edition license
my choice was effectively narrowed to the FPGA chips available under the WebPack version of Vivado. 

{{< figure
    src="xilinx_doc.png"
    alt="Xilinx supported boards per vivado edition" 
    caption="Xilinx supported boards per vivado edition" 
>}} 


Unsurprisingly Xilinx are well aware of how top of the range the Virtex series are, 
and doesn't offer any Virtex UltraScale+ chips with the webpack license. 
That said, they do offer support for two very respectable Kintex UltraScale+ FPGA models, the `XCKU3P` and the `XCKU5P`. [1]

{{< figure
    src="fpga_kintex.png"
    alt="Xiling product guide, overview for the Kintex UltraScale+ series"
    caption="Xiling product guide, overview for the Kintex UltraScale+ series"
>}}

These two chips are far from being small hobiest toys, with the smaller `XCUK3P` already boasting +162K LUTs and 
16 GTY transceivers, capable, depending on the physical constraints imposed by the chip packaging of 
operating at up to 32.75Gb/s.
  
Now that the chip selection has been narrowed down I set out to look for a dev board. 
 
My requirements for the board where that it featured : 
- at least 2 SFP+ or 1 QSFP connector 
- a JTAG interface 
- a PCIe interface at least x8 wide

As to where to get the board from, my options where : 
1. Design the board myself
2. Get the AXKU5 or AXKU3 from Alinx
3. See what I could unearth on the second hand market

Although option `1` could have likely been the most interesting, designing a 
dev board with both a high speed PCIe and ethernet interface was not the goal of 
today's project. 

As for option `2`,
Alinx is  newer vendor that is still building up it's credibility in the west, 
there technical documentation is a bit sparse, but people that have 
experimented with them seem to have experienced any issues.
Most importantly, Alinx provided very fairly priced development boards ranging
in the 900 to 1050 dollar ranges ( +150$ for the HPC FMC SFP+ extension board ).
Although these are not cheap by any metric, compared to the competition 
price point they are good value.


Option `2` was comming up ahead until I stumbled upon this ebay listing : 
 
{{< figure 
    src="ebay.png"
    alt="Ebay listing for a decommissioned Alibaba Cloud accelerator FPGA"
    caption="Ebay listing for a decommissioned Alibaba Cloud accelerator FPGA"
>}}
For 200$ this board featured a `XCKU3P-FFVB676`, 2 SPF+ connector and a x8 PCIe interface. 
On the flip side it came with no documentation whatsoever, no guaranty it worked, and the 
faint promise in the listing that there was a JTAG interface. 
A sane person would likely have dismissed this as an interesting internet oddity, a remanence 
of what happens when a generation of accelerator cards gets phased out in favor of the next, 
or maybe just an expensive paperweight. 

But I like a challenge, and the appeal of unlocking the 200$ Kintex UltraScale+ development board 
was to great to ignore. 

As such, I aim for this article to become the documentation paving the way to though this mirage. 

# The debugger challenge 

Xilinx outlines a list JTAG probes it supports for debugging and configuring there FPGA's. 
I do not personnally own any of these probes and am not looking to buy yet another vendor specific probe unless necessary. 
This does mean I will be scarificing the ability to interface with the very handy ILA (Integrated Logic
Analyzer). That being said nothing is stopping us from building our own ILA equivalent logic, and like the 
ILA reporting the captured information though JTAG via the available the JTAG user registers  

There is open source project called OpenOCD (Open On-Chip Debugger), it's aim is to provide debugging, in-system programming, 
and boundary-scan testing for a wide range of embedded targets through various JTAG/SWD adapters.
It is typically used for programming and debugging embedded SOCs.  

Given it's large library of already supported debug probes and boards, OpenOCB is common run by invoking pre-build configurations. 

That said, openOCD is acctually a very capable and configurable tool which allows us a great 
degree of control over what the JTAG adapter is going. Additionally, it has support out of the box for 
standard SVF(Serial Vector Format) format used to describe a sequence of JTAG operations and I know that 
Vivado can generate a SVF file.
As such, although there isn't really and documentation of configuring an xilinx FPGA past the 7 series, 
and most of the most recent developpement are focused more on it's ability to debug embeeded Zync arm cores, 
I figured it should be doable.

Wish me luck.   
 
# The plan 

So, to resume the current plan is to buy a second hand hardware accelerator of ebay at a too good to be true price, and try to configure it
with an unofficial probe using open source software without any clear official support.  
The awnser to the obvious question you are thinking if you, like me have been around the block a few times is: so many things can go wrong. 

As such, we need as to how to approach this. 
The goal of this plan is to outline incremental steps I can follow to build upon themselves with the end goal of being able to use this as a dev board. 

## 1 - Confirming the board works 
 
First order of buisness will be to confirm the board is showing signs of working as intended. 

There is a high probabiliy that the flash wasn't wipped before this board was sold off, as such the pervious bitstream should
still be in the flash. 
Given this board was used as an accelerator, we should be able to use that to confirm the board is working by either checking if 
the board is presenting itself as a PCIe endpoint or if the sfp's are sending the ethernet PHY idle sequence. 

## 2 - Connecting a debugger to it

Next step is to try and get the debugger connected.
The ebay listing advertized there is a JTAG interface, but the picture is grainy enoght that where that JTAG is and what pins are 
available is unclear. 

Additionally, we have no indication of what devices are daisy chainned together onto the JTAG scan chain. 
This is an essential question if we want to use JTAG for flashing, so we will need to figure that out. 

At this point, it would be strategic to use try and do some more probing into the FPGA via JTAG. 
Xilinx FPGA's 
exposes a hand full of usefull functionalities accessible over JTAG. The most well known of these is likely the 
SYSMON, which for example allows us to get real time temperature and voltage reading from inside the chip. 
Although openOCD doesn't have sysmon support out of the box it would be worth while to build it to : 
1. Familize myself with openOCD scripting, usefull for building my ILA replacement down the line and when debugging
2. Having an easy side channel to monitor FPGA operating parameters 
3. openOCD did have support for interfacing with the sysmon's ancestor, the XADC on the series 7, so it would be a nice contribution to add 

## 3 - Figuring out the pinnout 

The next hardest part will be figuring out the FPGA's pinnout and my clock sources. 
The biggest questions that need awnsering will be : 
- what external clocks sources do I have, what are there frequencies and which pins are they connected to 
- which transivers are the SFPs connected to 
- which transivers is the PCIe connected to

## 4 - Writing a bitstream 

For the time being I will be focusing on writing just temporary configurations over JTAG and not re-writing the flash. 

I plan on trying to write either the bitstream directly though openOCD's `virtex2` + `pld` drivers, or by following the 
SVF generated by Vivado. 
My hopes are higher for the SVF path, but since I will be generating a bitstream anyways in order to build my SVF file 
I will be trying both. 
Additionally, since I believe a low itteration time is paramount to project velocity and getting big things done, I also want automatize
all of the vivado flow from taking the rtl to the svf generation. 

Simple enogth, right ?


# Livness test

A few days latter my prize arrived via extress mail. 

{{< figure
    src="fpga.jpg"
    alt="fpga"
    caption="My prized Kintex UltraScale\+ FPGA board also know as the decomissioned alibaba cloud accelerator. Jammed transiver now safely removed."
>}}

Unexpectedly it even came with a free 25G SFP28 Huawei transceiver rated for a 300m distance and a single 1m long OS2 fiber patch cable. 
This might not have been intentional as the transceiver was jammed in the SFP cage, but it was still very generous of them to include the fiber patch cable.
After all who doesn't like free stuff?

{{< figure
    src="free_stuff.jpg"
    alt="Additional SFP28-25G-1310nm-300m-SM Huawei transiver, and 1m long OS2 patch cable" 
    caption="Free additional SFP28-25G-1310nm-300m-SM Huawei transiver, and 1m long OS2 patch cable" 
>}}

The board also came with a travel case and half of a PCIe to USB adapter and a 12V power supply that one could use to power the board as a standalone device. Although this standalone configuration will not be of any use to me, for those looking to develop just networking interfaces without any PCIe interface, this could come in handy.

Overall the board looked a little worn, but both the transceiver cages and PCIe connectors didn't look to be damaged.

## Standalone configuration 

Before real testing could start I first did a small power-up test using the PCIe to USB adapter that the seller provided. 

This allowed me to power up the board and using the LEDs on the board and the heat produced by the FPGA 
I was able to do a quick check that the board seemed to be powering up at a surface level (pun intended).

## PCIe interface

{{< alert >}}
As a reminder, this next test relied on the flash not having been wiped and someone else's design still
being loaded.
{{< /alert >}}

Since I didn't want to directly plug this into my prized build server, I decided to use a Raspberry Pi 5 as
my test device and got myself an external PCIe adapter.

It just so happened that the latest Raspberry Pi 5 now features an external PCIe Gen 2.0 x1 interface.
Though our FPGA can handle up to a PCIe Gen 3.0 and the board had a x8 wide interface,
since PCIe standard is backwards compatible and the number of lanes on the interface can be downgraded,
plugging our FPGA with this Raspberry Pi should work.

{{< figure 
    src="pi.jpg"
    alt="FPGA board connected to the Raspberry Pi 5 via the PCIe to PCIe x1 adapter"
    caption="FPGA board connected to the Raspberry Pi 5 via the PCIe to PCIe x1 adapter"
>}}

After both the Raspberry and the FPGA were booted, I SSHed into my rpi and
started looking for the PCIe enumeration sequence logged from the Linux
PCIe core subsystem.
 
`dmesg` log : 

```
[    0.388790] pci 0000:00:00.0: [14e4:2712] type 01 class 0x060400
[    0.388817] pci 0000:00:00.0: PME# supported from D0 D3hot
[    0.389752] pci 0000:00:00.0: bridge configuration invalid ([bus 00-00]), reconfiguring
[    0.495733] brcm-pcie 1000110000.pcie: link up, 5.0 GT/s PCIe x1 (!SSC)
[    0.495759] pci 0000:01:00.0: [dabc:1017] type 00 class 0x020000
```

### Background infomration 

Since most people might not be intimatly as familiar with PCIe terminology, allow me to 
take a step back and give more details as to what is going on here. 

`0000:00:00.0`: is the identifier of a specific PCIe device connected through the PCIe network 
to the kernel, it read as `domain`:`bus`:`device`.`function`.

`[14e4:2712]`: is the device's `[vendor id:device id]`, these vendor id identifiers are 
assigned by the PCI standard body to hardware vendors. Vendors are then free to define there 
own vendor id's. The full list of official vendor id's and released device id can be found : https://admin.pci-ids.ucw.cz/read/PC/14e4 or in
a less user friendly format in the linux kernel code : https://github.com/torvalds/linux/blob/7aac71907bdea16e2754a782b9d9155449a9d49d/include/linux/pci_ids.h#L160-L3256

`type 01`: PCIe has two trypes of devices, bridges allowing the connection of multiple downstream devices to an 
upstream device, and endpoints.
Bridges are of type `01` and endpoints of type `00`.

`class 0x60400`: is the PCIe device class and catheogrises what kind of function this device performs. It 
has the following format `0x[Base Class (8 bits)][Sub Class (8 bits)][Programming Interface (8 bits)]`, 
though the nasty little secret is that the sub class field is often ignored. 
A list of class and sub class idenfiers can be found: https://admin.pci-ids.ucw.cz/read/PD or in the linux codebase : https://github.com/torvalds/linux/blob/7aac71907bdea16e2754a782b9d9155449a9d49d/include/linux/pci_ids.h#L15-L158

### Dmesg log 

Of our dmesg content the follow two lines are the most relevant : 
```
[    0.388790] pci 0000:00:00.0: [14e4:2712] type 01 class 0x060400
[    0.495759] pci 0000:01:00.0: [dabc:1017] type 00 class 0x020000
```

Firstly the PCIe subsystem is logging at `0000:00:00.0` it has discovered a Broadcom ( vendor id `14e4` ) BCM2712 PCIe Bridge ( device id `0x2712` ). 
As the name suggests and the type of `01` confirms this is a bridge, the class of `0x0604xx` tells us exactly what it bridges to. 
Here it is a PCI-to-PCI bridge, meaning that more devices can be connected downstream of it, as such, the search continues. 

The subsystem then discovers a second device at `0000:01:00.0`, this is an endpoint device as indicated by its type of `00`
and its class of `0x02000` tells us this is ethernet networking equipment. 
Our next hint is the `dabc` doesn't correspond to a known vendor id. When designing a PCIe interface in hardware these 
are parameters we can set. Additionally, among the different ways Linux uses to identify which driver to load for a PCIe device 
the vendor id and device id can be used. Supposing we are implementing custom logic, in order to prevent any bug where the wrong driver 
could be loaded, it is best to use a separate vendor id. 
This also helps identify your custom accelerator at a glance. 

As such, it is not surprising to see an unknown vendor id appear for 
an FPGA, this with the class as an ethernet networking device give us strong indication this is our board.

### Full PCIe device status

The dmesg logs gave us a good overview of the situation but for additional details we can turn to `lspci`.
The most verbose output gives us a full overview of the devices capabilities and current configuration.


Broadcom bridge: 
```
0000:00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2712 PCIe Bridge (rev 21) (prog-if 00 [Normal decode])
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 38
        Bus: primary=00, secondary=01, subordinate=01, sec-latency=0
        Memory behind bridge: [disabled] [32-bit]
        Prefetchable memory behind bridge: 1800000000-182fffffff [size=768M] [32-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR- NoISA- VGA- VGA16- MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [48] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=1 PME-
        Capabilities: [ac] Express (v2) Root Port (Slot-), MSI 00
                DevCap: MaxPayload 512 bytes, PhantFunc 0
                        ExtTag- RBE+
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr+ NoSnoop+
                        MaxPayload 512 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #0, Speed 5GT/s, Width x1, ASPM L0s L1, Exit Latency L0s <2us, L1 <4us
                        ClockPM+ Surprise- LLActRep- BwNot+ ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 5GT/s, Width x1
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt+
                RootCap: CRSVisible+
                RootCtl: ErrCorrectable- ErrNon-Fatal- ErrFatal- PMEIntEna+ CRSVisible+
                RootSta: PME ReqID 0000, PMEStatus- PMEPending-
                DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via WAKE#, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- LN System CLS Not Supported, TPHComp- ExtTPHComp- ARIFwd+
                         AtomicOpsCap: Routing- 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled, ARIFwd-
                         AtomicOpsCtl: ReqEn- EgressBlck-
                LnkCap2: Supported Link Speeds: 2.5-5GT/s, Crosslink- Retimer- 2Retimers- DRS+
                LnkCtl2: Target Link Speed: 5GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
                         EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported, DRS-
                         DownstreamComp: Link Up - Present
        Capabilities: [100 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                AERCap: First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
                RootCmd: CERptEn+ NFERptEn+ FERptEn+
                RootSta: CERcvd- MultCERcvd- UERcvd- MultUERcvd-
                         FirstFatal- NonFatalMsg- FatalMsg- IntMsg 0
                ErrorSrc: ERR_COR: 0000 ERR_FATAL/NONFATAL: 0000
        Capabilities: [160 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=1
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=00 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64- WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=Fixed TC/VC=ff
                        Status: NegoPending- InProgress-
        Capabilities: [180 v1] Vendor Specific Information: ID=0000 Rev=0 Len=028 <?>
        Capabilities: [240 v1] L1 PM Substates
                L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1+ ASPM_L1.2+ ASPM_L1.1+ L1_PM_Substates+
                          PortCommonModeRestoreTime=8us PortTPowerOnTime=10us
                L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
                           T_CommonMode=1us LTR1.2_Threshold=0ns
                L1SubCtl2: T_PwrOn=10us
        Capabilities: [300 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Kernel driver in use: pcieport
```
FPGA board: 
```
0000:01:00.0 Ethernet controller: Device dabc:1017
        Subsystem: Red Hat, Inc. Device a001
        Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Region 0: Memory at 1820000000 (64-bit, prefetchable) [disabled] [size=2K]
        Region 2: Memory at 1800000000 (64-bit, prefetchable) [disabled] [size=512M]
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [70] Express (v2) Endpoint, MSI 00
                DevCap: MaxPayload 1024 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
                        ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0W
                DevCtl: CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
                        RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 512 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #0, Speed 8GT/s, Width x8, ASPM not supported
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 5GT/s (downgraded), Width x1 (downgraded)
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Range BC, TimeoutDis+ NROPrPrP- LTR-
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled,
                         AtomicOpsCtl: ReqEn-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
                         EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [100 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                AERCap: First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap- ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [1c0 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
```

The `lspci` logs give us a lot of useful information on the current status of PCIe devices, 
but I would like to call your focus on the following particularly interesting lines reported 
for our FPGA : 
```
                LnkCap: Port #0, Speed 8GT/s, Width x8, ASPM not supported
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 5GT/s (downgraded), Width x1 (downgraded)0x060400
```
The `LnkCap` tells us about the full capabilities of this PCIe device, here we can see that 
the current design is a PCIe Gen 3.0 x8. 
That said the `LnkSta` tells us our speed has been downgraded to that of PCIe Gen 2.0 at 5GT/s and 
that our width is only x1. 

When a new PCIe device is plugged in or during startup, PCIe performs a link speed and width negotiation 
where it tries to reach the highest supported stable configuration for the current system. 
In our current system, although our FPGA is capable of 8GT/s, since it is located downstream of the 
Broadcom bridge with a maximum link capacity of Gen 2.0, the FPGA has been downgraded to 5GT/s.

As for the width of x1, that is expected since the Broadcom bridge is also only x1 wide, and the other 
7 PCIe lanes are literally hanging over the side. 

{{< figure
    src="pcie_air.jpg"
    alt="7 PCIe lanes left unconnected and hangging over the air"
    caption="7 PCIe lanes left unconnected and hangging over the air"
>}}

Thanks to this we can confirm that our board seems to be working, and we can now move to figuring 
out how to get the JTAG connection. 

# JTAG interface 

Xilinx FPGAs can be configured by writing a bitstream to their internal CMOS Configuration Latches (CCL).
This memory is volatile, so this configuration must be re-done on every power cycle.
For in the field devices this bitstream would typically be read from an external SPI memory during initialization,
or written from an external device, like an embedded controller, but for development purposes overwriting the contents of the CCLs over JTAG is acceptable.

This configuration is done by shifting in the entire FPGA configuration bitstream into the JTAG bus.

## FPGA board JTAG interface 

As promised by the original eBay listing the board did come with an accessible JTAG interface, and there
wasn't even the need for any additional soldering.

{{< figure
    src="pcb_jtag.jpg"
    alt="View of the JTAG interface on the PCB"
    caption="View of the JTAG interface on the PCB"
>}} 


In addition to a power reference, and ground, it featured the four mandatory signals comprising the JTAG TAP, 
which are : 
- **TCK** Test Clock 
- **TMS** Test Mode Select
- **TDI** Test Data Input 
- **TDO** Test Data Output 

The JTAG interface can also come with an independent reset signal. 
That said, since Xilinx JTAG interfaces do not have this independent reset signal, we will need to use the JTAG FSM reset state
as our reset signal.

{{< figure 
    src="board_jtag_intf.svg"
    alt="very nice documentation of the board jtag pinout"
    caption="6 pin board jtag interface"
>}}

Another issue with this layout is that, likely in the interest of saving on space and
manufacturing cost given this accelerator was not designed as a dev board, this JTAG interface
doesn't follow an easily compatible layout on which I can just plug in one of my debug probes.
As such, it will require some re-wiring.

## Segger JLINK :heart: 

I do not own an AMD approved JTAG programmer. 

Traditionally speaking, the Segger JLink is used for debugging embedded CPUs let them be standalone or in a
Zynq, rather than for configuring FPGAs.

That being said, all we need to do is use JTAG to shift in a bitstream to the CCLs and
technically speaking any programmable device with 4 sufficiently fast GPIOs can be used as a JTAG programmer.
Additionally, the JLink is well supported by OpenOCD, the JLink's libraries are open source, and I happened to own one.

{{< alert icon="circle-info" >}} 
Note : I could also have used a USB Blaster, which considering it is literally an Altera tool would have made it hilarious.
{{< /alert >}}
{{< figure 
    src="segger_jlink_conn.svg"
    alt="very nice 20 pin segger jlink pinnout interface documentation"
    caption="20 pin segger jlink pinnout"
    >}}

### Wiring

Given my PCB's JTAG interface wasn't out of the box compatible with my JLink's probe
it required some small rewiring.

{{< figure
    src="jtag_wiring.svg"
    alt="very nice jtag wiring driagram to connect jlink jtag probe to fpga board"
    caption="wiring driagram to connect jlink jtag probe to fpga board"
>}}

JTAG is a parallel protocol where `TDI` and `TMS` will be captured on the `TCK` rising edge. 
Because of this, good JTAG PCB trace length matching is advised in order to minimize skew. 

{{< figure
    src="jtag_timing.png"
    alt="Timing Waveform for JTAG Signals (From Target Device Perspective)"
    caption="Timing Waveform for JTAG Signals (From Target Device Perspective); source : https://www.intel.com/content/www/us/en/docs/programmable/683719/current/jtag-timing-constraints-and-waveforms.html"
>}} 

Ideally a custom connector with length matched traces to work as an interface between the JLink's
probe and a board specific connector would be used. This could be a 20 minute KiCad project
and be back in under a week using OSH Park or JLCPCB.

{{< figure
    src="con.jpg"
    alt="Far from length matched JTAG connections" 
    caption="Far from length matched JTAG connections" 
>}}

And yet, here we are shoving breadboard wires between our debugger and the board.
On the flip side, we can increase the skew tolerance by slowing down the TCK clock signal and
it just so happens that OpenOCD allows us to easily control the debugger clock speed. As such
there is no need for a custom connector and we can work around this.

{{< alert >}}
If no clock speed is specified OpenOCD sets the clock speed at 100MHz. 
This is to high for our case. 
As such, latter in the article, I will be setting the JTAG clock down to 1MHz for probing and 
programming will be done at 10MHz.
Both speeds show no issues. 
{{< /alert >}}

## OpenOCD

OpenOCD is a free and open source on-chip debugger software that aims to be compatible with as many
probes, boards and chips as possible.

Since OpenOCD has support for the standard SVF file format, my plan for my flashing flow will be to use
Vivado to generate the SVF and have OpenOCD flash it.
Now, some of you might be starting to notice that I am diverging quite far from the well lit path of officially
supported tools. Not only am I using a not officially supported debug probe, but I am also using some
obscure open source software with questionable support for interfacing with Xilinx UltraScale+ FPGAs.
You might be wondering, given that the officially supported tools can already prove themselves to be a headache to get working properly,
why I am seemingly making my life even harder?

The reason is quite simple: when things inevitably start going wrong, as they will given the nature of the project,
having an entirely open toolchain where all the code is accessible and modifiable, allows me to have more visibility
as to what is going on.
I cannot delve into a black box in the same fashion.

### Building OpenOCD

By default the version of OpenOCD that I got on my test server via the packet manager was quite outdated and missing features 
I will need down the line. 

Additionally, given it was unclear if configuring an Xilinx UltraScale+ FPGA's
had every been attemplted, due to the absence of posts on the matter, I figured I might run into a few issues
and having the ability to modify OpenOCB's source code could come in handy. 
 
As such, I decided to re-build it from source. 

This explains why, in following logs, I will be running OpenOCD version `0.12.0+dev-02170-gfcff4b712`.

Note : I have additionally re-build the jlink libs from source. 

## Determining the scan chain 

Since I do not have the shematics for the board I do not know how many devices are daisy-chainned on the board JTAG BUS. 
Additionally, I would like to confirm if the FPGA on the ebay listing is actually the one on the board. 
In the JTAG standard, each chainned device shall expose an accessible `IDCODE` register. 
This register is used to identify the manifacturer, device type, and revision number. 

By default, when setting up the JTAG server, one is expected to configure the TAPs on the scan chain with the expected `IDCODE` values
and the length of the instruction register for each device. 
Given this is an undocumented board off eaby, I am not sure what the chain looks like. 
Fortunatly, OpenOCB has an autoprobing functionallity, where it will do a bling interrogation in an **attempt** to discover 
the available TAPs and report them out. 

As such, my first order of buisness was doing this autoprobing. 

I used the following OpenOCB configuration, the autoprobling will be used as I did not specify any taps. 

```tcl
source [find interface/jlink.cfg]
transport select jtag

set SPEED 1
jtag_rclk $SPEED
adapter speed $SPEED

reset_config none
```

The blind interrogation sucessfully discovered a single TAP on the chain with an `IDCODE` of `0x04a63093`. 

```
gp@workhorse:~/tools/openocd_jlink_test/autoprob$ openocd
Open On-Chip Debugger 0.12.0+dev-02170-gfcff4b712 (2025-09-04-21:02)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
none separate
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : J-Link V10 compiled Jan 30 2023 11:28:07
Info : Hardware version: 10.10
Info : VTarget = 1.812 V
Info : clock speed 1 kHz
Warn : There are no enabled taps.  AUTO PROBING MIGHT NOT WORK!!
Info : JTAG tap: auto0.tap tap/device found: 0x04a63093 (mfg: 0x049 (Xilinx), part: 0x4a63, ver: 0x0)
Warn : AUTO auto0.tap - use "jtag newtap auto0 tap -irlen 2 -expected-id 0x04a63093"
Error: IR capture error at bit 2, saw 0x3ffffffffffffff5 not 0x...3
Warn : Bypassing JTAG setup events due to errors
Warn : gdb services need one or more targets defined
```

Comparing against the `UltraScale Architecture Configuration User Guide (UG570)` we see that this `IDCODE` matches up
perfectly with the expected value for the `KU3P`. 

{{< figure
    src="idcode.png"
    alt="JTAG and IDCODE for UltraScale Architecture-based FPGAs"
    caption="JTAG and IDCODE for UltraScale Architecture-based FPGAs"
>}}

By default OpenOCB assumes a JTAG instruction lenght of 2 bits while our FPGA actually have an IR length of 6 bits. 
This is the root cause behind the IR capture error encountered during autoprobing `JTAG and IDCODE for UltraScale Architecture-based FPGAs`.
We can confirm this was our issues as, when we update our simple probling script to determine the `IDCODE` of a single 
TAP with an IR length of 6 bits we can re-detert the FPGA with no additional errors. 

```tcl
source [find interface/jlink.cfg]
transport select jtag

set SPEED 1
jtag_rclk $SPEED
adapter speed $SPEED

reset_config none

jtag newtap auto_detect tap -irlen 6
```

Output : 
```
gp@workhorse:~/tools/openocd_jlink_test/autoprob$ openocd
Open On-Chip Debugger 0.12.0+dev-02170-gfcff4b712 (2025-09-04-21:02)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : J-Link V10 compiled Jan 30 2023 11:28:07
Info : Hardware version: 10.10
Info : VTarget = 1.812 V
Info : clock speed 1 kHz
Info : JTAG tap: auto_detect.tap tap/device found: 0x04a63093 (mfg: 0x049 (Xilinx), part: 0x4a63, ver: 0x0)
Warn : gdb services need one or more targets defined
```

Thus based on the probing, this is the JTAG scan chain for our board : 

{{< figure 
    src="scan_chain.svg"
    alt="JTAG scan chain for the alibaba cloud FPGA"
    caption="JTAG scan chain for the alibaba cloud FPGA"
>}}

## Systerm Monitor Registers

Previous generations of Xilinx FPGA had a system called the XADC that, among other things, 
allowed you to acquire chip temperature and voltage readings. The newer UltraScale and UltraScale+ 
family have deprecated this XADC module in favor of the SYSMON (and SYSMON4) which allows you to also 
get these temperature readings but better.

Unfortunately, openOCD didn't have support for reading the SYSMON over JTAG out of the box, so I will be adding it.

{{< alert icon="circle-info" >}}

To be more precise, the Kintex UltraScale+ has a SYSMON4 and not a SYSMON. 
For full context, there are 3 flavors of SYSMON:

- `SYSMON1` used in the Kintex and Virtex UltraScale series
- `SYSMON4` used in the Kintex, Virtex and in the Zynq programmable logic for the UltraScale+ series 
- `SYSMON` used in the Zynq in the processing system of the UltraScale+ series. \
Yes, you read that correctly the Zynq of the UltraScale+ series feature not one, but at least two unique sysmon instances. 

Ultimately, for the purpose of this article, all these instances are similar enough that I will be using the terms SYSMON4 and SYSMON interchangeably.

{{< /alert >}} 

In order for the JTAG to interact with the SYSMON, we first need to write the `SYSMON_DRP` command to the 
JTAG Instruction Register (IR). 
Looking at the documentation, we see that this command has a value of `0x37`. Funnily enough, 
this has the same command code as the XADC, solidifying the SYSMON as the XADC's descendant.

The SYSMON offers a lot more additional functionalities than just being used to read voltage and temperature, 
but for today's use case we will not be using any of that. Rather, we will focus only on reading a 
subset of the SYSMON status registers.

These status registers are located at addresses `(00h-3Fh, 80h-BFh)`, 
and contain the measurement results of the analog-to-digital conversions, the flag registers,
 and the calibration coefficients. We can select which address we wish to read by writing the 
address to the Data Register (DR) over JTAG and the data will be read out of `TDO`.

I then added this sequence to read the current chip temperature, internal and external 
voltages as well as the maximum values for these recorded since FPGA power cycle, to my flashing script output:
```
gp@workhorse:~/tools/openocd_jlink_test$ openocd
Open On-Chip Debugger 0.12.0+dev-02170-gfcff4b712 (2025-09-04-20:02)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
set chipname XCKU3P
Read temperature sysmon 4
Info : J-Link V10 compiled Jan 30 2023 11:28:07
Info : Hardware version: 10.10
Info : VTarget = 1.819 V
Info : clock speed 1 kHz
Info : JTAG tap: XCKU3P.tap tap/device found: 0x04a63093 (mfg: 0x049 (Xilinx), part: 0x4a63, ver: 0x0)
Warn : gdb services need one or more targets defined
--------------------
Sysmon status report :
TEMP 31.12 C
MAXTEMP 34.62 C
VCCINT 0.852 V
MAXVCC 0.855 V
VCCAUX 1.805 V
MAXVCCAUX 1.807 V
```
These readings seem coherent and the external voltage closely resembles the voltage the debug probe is recording.

# Pinout 


To my indescribable joy I happened to stumble onto this gold mine, in which we get the board pinout.
This most likely fell off a truck: https://blog.csdn.net/qq_37650251/article/details/145716953 
(If you cannot access the full article, inspect the html source)

This gives us the following pinout. 
So far this pinout looks correct and I haven't spotted any glaring issues with it.


| Pin Index | Name | IO Standard | Location | Bank |
|-----------|------|-------------|----------|------|
| 0 | diff_100mhz_clk_p | LVDS | E18 | BANK67 |
| 1 | diff_100mhz_clk_n | LVDS | D18 | BANK67 |
| 2 | sfp_mgt_clk_p | LVDS | K7 | BANK227 |
| 3 | sfp_mgt_clk_n | LVDS | K6 | BANK227 |
| 4 | sfp_1_txn | - | B6 | BANK227 |
| 5 | sfp_1_txp | - | B7 | BANK227 |
| 6 | sfp_1_rxn | - | A3 | BANK227 |
| 7 | sfp_1_rxp | - | A4 | BANK227 |
| 8 | sfp_2_txn | - | D6 | BANK227 |
| 9 | sfp_2_txp | - | D7 | BANK227 |
| 10 | sfp_2_rxn | - | B1 | BANK227 |
| 11 | sfp_2_rxp | - | B2 | BANK227 |
| 12 | SFP_1_MOD_DEF_0 | LVCMOS18 | D14 | BANK87 |
| 13 | SFP_1_TX_FAULT | LVCMOS18 | B14 | BANK87 |
| 14 | SFP_1_LOS | LVCMOS18 | D13 | BANK87 |
| 15 | SFP_1_LED | LVCMOS18 | B12 | BANK87 |
| 16 | SFP_2_MOD_DEF_0 | LVCMOS18 | E11 | BANK86 |
| 17 | SFP_2_TX_FAULT | LVCMOS18 | F9 | BANK86 |
| 18 | SFP_2_LOS | LVCMOS18 | E10 | BANK86 |
| 19 | SFP_2_LED | LVCMOS18 | C12 | BANK87 |
| 20 | IIC_SDA_SFP_1 | LVCMOS18 | C14 | BANK87 |
| 21 | IIC_SCL_SFP_1 | LVCMOS18 | C13 | BANK87 |
| 22 | IIC_SDA_SFP_2 | LVCMOS18 | D11 | BANK86 |
| 23 | IIC_SCL_SFP_2 | LVCMOS18 | D10 | BANK86 |
| 24 | IIC_SDA_EEPROM_0 | LVCMOS18 | G10 | BANK86 |
| 25 | IIC_SCL_EEPROM_0 | LVCMOS18 | G9 | BANK86 |
| 26 | IIC_SDA_EEPROM_1 | LVCMOS18 | J15 | BANK87 |
| 27 | IIC_SCL_EEPROM_1 | LVCMOS18 | J14 | BANK87 |
| 28 | GPIO_LED_R | LVCMOS18 | A13 | BANK87 |
| 29 | GPIO_LED_G | LVCMOS18 | A12 | BANK87 |
| 30 | GPIO_LED_H | LVCMOS18 | B9 | BANK86 |
| 31 | GPIO_LED_1 | LVCMOS18 | B11 | BANK86 |
| 32 | GPIO_LED_2 | LVCMOS18 | C11 | BANK86 |
| 33 | GPIO_LED_3 | LVCMOS18 | A10 | BANK86 |
| 34 | GPIO_LED_4 | LVCMOS18 | B10 | BANK86 |
| 35 | pcie_mgt_clkn | - | T6 | BANK225 |
| 36 | pcie_mgt_clkp | - | T7 | BANK225 |
| 37 | pcie_tx0_n | - | R4 | BANK225 |
| 38 | pcie_tx1_n | - | U4 | BANK225 |
| 39 | pcie_tx2_n | - | W4 | BANK225 |
| 40 | pcie_tx3_n | - | AA4 | BANK225 |
| 41 | pcie_tx4_n | - | AC4 | BANK224 |
| 42 | pcie_tx5_n | - | AD6 | BANK224 |
| 43 | pcie_tx6_n | - | AE8 | BANK224 |
| 44 | pcie_tx7_n | - | AF6 | BANK224 |
| 45 | pcie_rx0_n | - | P1 | BANK225 |
| 46 | pcie_rx1_n | - | T1 | BANK225 |
| 47 | pcie_rx2_n | - | V1 | BANK225 |
| 48 | pcie_rx3_n | - | Y1 | BANK225 |
| 49 | pcie_rx4_n | - | AB1 | BANK224 |
| 50 | pcie_rx5_n | - | AD1 | BANK224 |
| 51 | pcie_rx6_n | - | AE3 | BANK224 |
| 52 | pcie_rx7_n | - | AF1 | BANK224 |
| 53 | pcie_tx0_p | - | R5 | BANK225 |
| 54 | pcie_tx1_p | - | U5 | BANK225 |
| 55 | pcie_tx2_p | - | W5 | BANK225 |
| 56 | pcie_tx3_p | - | AA5 | BANK225 |
| 57 | pcie_tx4_p | - | AC5 | BANK224 |
| 58 | pcie_tx5_p | - | AD7 | BANK224 |
| 59 | pcie_tx6_p | - | AE9 | BANK224 |
| 60 | pcie_tx7_p | - | AF7 | BANK224 |
| 61 | pcie_rx0_p | - | P2 | BANK225 |
| 62 | pcie_rx1_p | - | T2 | BANK225 |
| 63 | pcie_rx2_p | - | V2 | BANK225 |
| 64 | pcie_rx3_p | - | Y2 | BANK225 |
| 65 | pcie_rx4_p | - | AB2 | BANK224 |
| 66 | pcie_rx5_p | - | AD2 | BANK224 |
| 67 | pcie_rx6_p | - | AE4 | BANK224 |
| 68 | pcie_rx7_p | - | AF2 | BANK224 |
| 69 | pcie_perstn_rst | LVCMOS18 | A9 | BANK86 |

## Global clock 

On an UltraScale+, high-speed global clocks are typically driven from external sources, 
often using differential pairs for better signal integrity.


According to the pinout we have two such differential pairs.

My first order of business is determining which of these two I can use to easily drive my global clocks.

These differential pairs are provided over the following pins:
- 100MHz : {E18, D18} 
- 156.25MHz : {K7, K6} 


Judging by the naming and the frequencies, the 156.25MHz clock is likely my SFP reference clock, 
and the 100MHz can be used as my global clock. 

We can confirm by querying the pin properties.

**K6** properties : 
```
Vivado% report_property [get_package_pins K6]
Property                Type    Read-only  Value
BANK                    string  true       227
BUFIO_2_REGION          string  true       TR
CLASS                   string  true       package_pin
DIFF_PAIR_PIN           string  true       K7
IS_BONDED               bool    true       1
IS_DIFFERENTIAL         bool    true       1
IS_GENERAL_PURPOSE      bool    true       0
IS_GLOBAL_CLK           bool    true       0
IS_LOW_CAP              bool    true       0
IS_MASTER               bool    true       0
IS_VREF                 bool    true       0
IS_VRN                  bool    true       0
IS_VRP                  bool    true       0
MAX_DELAY               int     true       38764
MIN_DELAY               int     true       38378
NAME                    string  true       K6
PIN_FUNC                enum    true       MGTREFCLK0N_227
PIN_FUNC_COUNT          int     true       1
PKGPIN_BYTEGROUP_INDEX  int     true       0
PKGPIN_NIBBLE_INDEX     int     true       0
```

**E18** properties : 
```
Vivado% report_property [get_package_pins E18]
Property                Type    Read-only  Value
BANK                    string  true       67
BUFIO_2_REGION          string  true       TL
CLASS                   string  true       package_pin
DIFF_PAIR_PIN           string  true       D18
IS_BONDED               bool    true       1
IS_DIFFERENTIAL         bool    true       1
IS_GENERAL_PURPOSE      bool    true       1
IS_GLOBAL_CLK           bool    true       1
IS_LOW_CAP              bool    true       0
IS_MASTER               bool    true       1
IS_VREF                 bool    true       0
IS_VRN                  bool    true       0
IS_VRP                  bool    true       0
MAX_DELAY               int     true       87126
MIN_DELAY               int     true       86259
NAME                    string  true       E18
PIN_FUNC                enum    true       IO_L11P_T1U_N8_GC_67
PIN_FUNC_COUNT          int     true       2
PKGPIN_BYTEGROUP_INDEX  int     true       8
PKGPIN_NIBBLE_INDEX     int     true       2
```
We can now confirm the following items:
* The differential pairings are correct: {K6, K7}, {E18, D18}
* We can easily use the 100MHz as a source to drive our global clocking network
* The 156.25MHz clock is to be used as the reference clock for our GTY transceivers and lands on bank 227 as indicated by the `PIN_FUNC` property `MGTREFCLK0N_227`
* We cannot directly use the 156.25MHz clock to drive our global clock network

With all this we have sufficient information to write a constraint file (`xdc`) for this board. The next challenge is getting the bitstream onto the FPGA.

# Writing the bitstream

My personal belief is that one of the most important contributors to design quality is iteration cost. 
The lower your iteration cost, the higher your design quality is going to be given the same amount of people and time.

As such I will invest the small upfront cost to have the workflow be as streamlined as efficiently feasible.

Thus, my workflow evolved into doing practically everything over 
the command line interfaces and only interacting with the tools, Vivado in this case, through tcl scripts.

## Vivado flow 


The goal of this flow is to, given a few verilog design and constraint files produce a SVF file. 
We will breaking this down into 3 steps : 
1. creating the vivado project
2. running the implementation 
3. generating the bitstream and the SVF 

Although I recognist this isn't a widespead practice for hardware projects, 
I will be using a makefile order to corrdinate and manage the dependancies between the different step.

We will be invoking vivado in batch mode, this allows us to provide a tcl script alongside script arguments, the 
format is as following : 

```bash
vivado -mode batch <path to tcl script> -tclargs <script args>
```

Although this allows us to easily break down our flow into incremental stages, proceeding in this manner has the 
drawback of restarting vivado and needing to re-load the project or the project checkpoint on each invokation. 

As such, as the project size and complexity grows so will the project load time, so segmenting the
flow into a large number of independant scripts and invoking them in batch mode does come at an increasing cost. 
But for small scale projects this added flexibility doesn't come at a significiant cost. 

I will not go though the entire build flow in detail and will directly jump towards genreating the SVF file. 

### Generating the SVF file

The SVF for Serial Vector Format is a human readable, vendor agnositc speficiation used to specify JTAG bus operations.
 
Example SVF file containing a simple test program: 
```svf 
! Initialize UUT
STATE RESET;
! End IR scans in DRPAUSE
ENDIR DRPAUSE;
! End DR scans in DRPAUSE
ENDDR DRPAUSE;
! 24 bit IR header
HIR 24 TDI (FFFFFF);
! 3 bit DR header
HDR 3 TDI (7);
! 16 bit IR trailer
TIR 16 TDI (FFFF);
! 2 bit DR trailer
TDR 2 TDI (3);
! 8 bit IR scan, load BIST opcode
SIR 8 TDI (41) TDO (81) MASK (FF);
! 16 bit DR scan, load BIST seed
SDR 16 TDI (ABCD);
! RUNBIST for 95 TCK Clocks
RUNTEST 95 TCK ENDSTATE IRPAUSE;
! 16 bit DR scan, check BIST status
SDR 16 TDI (0000) TDO(1234) MASK(FFFF);
! Enter Test-Logic-Reset
STATE RESET;
! End Test Program
``` 

Vivado has the possibility of generating a hardware aware SVF file for configuring our FPGA, allowing us to program
it independantly of any vendor tooling.

Since the SVF file litterally contains the bitstream written in clear hexademical, in the file, our first step is to generate
our design's bitstream. 

Vivado proper isn't the software that generates the SVF file, rather this task is done by the hardware manager,
this program handles the all flashing sequences.
  
Using the tcl commenad line we can launch a new instance `open_hw_manager` and connect to it `connect_hw_server`. 
Then, since JTAG is a daisy chainned bus, and given the SVF file is just a standardised way of specifying 
JTAG bus operations, in order to genreate a correct JTAG configuratoin sequence, we must inform the hardware manger 
of what our JTAG chain looks like. 

Thanks to our ealier probing of the scan chain, have established that out FPGA is the only device on the chain. 
To inform the hardware manager we must create a new device configureation ( the term "device" refers to the "board"
in this case ) and add our fpga to the chain using the `create_hw_device -part <device name>`. If we had multiple
devices we should register them following the order they appear on the chain. 

Finally to genereate the svf file, we must first select the device we wish to program `program_hw_device <hw_device>`, in our case we will select the
device we have just created, the we can write out the svf to the file using `write_hw_svf <path to svf file>`.

```tcl
set checkpoint_path [lindex $argv 0]
set out_dir [lindex $argv 1]
puts "SVF generation script called with checkpoint path $checkpoint_path, generating to $out_dir"

open_checkpoint $checkpoint_path

# defines
set hw_target "alibaba_board_svf_target"
set fpga_device "xcku3p"
set bin_path "$out_dir/[current_project]"

write_bitstream "$bin_path.bit" -force

open_hw_manager

# connect to hw server with default config
connect_hw_server
puts "connected to hw server at [current_hw_server]"

create_hw_target $hw_target
puts "current hw target [current_hw_target]"

open_hw_target

# single device on scan chain
create_hw_device -part $fpga_device
puts "scan chain : [get_hw_devices]"

set_property PROGRAM.FILE "$bin_path.bit" [get_hw_device]

#select device to program
program_hw_device [get_hw_device]

# generate svf file
write_hw_svf -force "$bin_path.svf"

close_hw_manager
exit 0
```

## Configuraing the FPGA using OpenOCD 

At this point we have both a raw bitstream and the bitstream wrapped with the programming sequence 
in the SVF file. 

Although not very widespread openOCD has a very nice `svf` replay command :

```
18.1 SVF: Serial Vector Format

The Serial Vector Format, better known as SVF, is a way to represent JTAG test patterns
in text files. In a debug session using JTAG for its transport protocol, OpenOCD supports
running such test files.

[Command]svf filename [-tap tapname] [[-]quiet] [[-]nil] [[-]progress]
[[-]ignore_error]

This issues a JTAG reset (Test-Logic-Reset) and then runs the SVF script from
filename.
Arguments can be specified in any order; the optional dash doesn’t affect their se-
mantics.

Command options:
− -tap tapname ignore IR and DR headers and footers specified by the SVF file
with HIR, TIR, HDR and TDR commands; instead, calculate them automatically
according to the current JTAG chain configuration, targeting tapname;
− [-]quiet do not log every command before execution;
− [-]nil “dry run”, i.e., do not perform any operations on the real interface;
− [-]progress enable progress indication;
− [-]ignore_error continue execution despite TDO check errors.
``` 

We invoke it in our openOCD script as : 
```
svf $svf_path -progress
```

Full flashing sequence log : 

```
gp@workhorse:~/tools/openocd_jlink_test$ openocd
Open On-Chip Debugger 0.12.0+dev-02170-gfcff4b712 (2025-09-04-21:02)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
set chipname XCKU3P
Read temperature sysmon 4
Info : J-Link V10 compiled Jan 30 2023 11:28:07
Info : Hardware version: 10.10
Info : VTarget = 1.812 V
Info : clock speed 1 kHz
Info : JTAG tap: XCKU3P.tap tap/device found: 0x04a63093 (mfg: 0x049 (Xilinx), part: 0x4a63, ver: 0x0)
Warn : gdb services need one or more targets defined
--------------------
Sysmon status report :
TEMP 50.46 C
MAXTEMP 52.79 C
VCCINT 0.846 V
MAXVCC 0.860 V
VCCAUX 1.799 V
MAXVCCAUX 1.809 V
--------------------
svf processing file: "out/project_prj_checkpoint.svf"
  0%  TRST OFF;
  0%  ENDIR IDLE;
  0%  ENDDR IDLE;
  0%  STATE RESET;
  0%  STATE IDLE;
  0%  FREQUENCY 1.00E+07 HZ;
adapter speed: 10000 kHz
  0%  HIR 0 ;
  0%  TIR 0 ;
  0%  HDR 0 ;
  0%  TDR 0 ;
  0%  SIR 6 TDI (09) ;
  0%  SDR 32 TDI (00000000) TDO (04a63093) MASK (0fffffff) ;
  0%  STATE RESET;
  0%  STATE IDLE;
  0%  SIR 6 TDI (0b) ;
  0%  SIR 6 TDI (14) ;
  0%  RUNTEST 0.100000 SEC;
  0%  RUNTEST 10000 TCK;
  0%  SIR 6 TDI (14) TDO (11) MASK (31) ;
  0%  SIR 6 TDI (05) ;
 95%  ffffffffffff) ;
 95%  SIR 6 TDI (09) TDO (31) MASK (11) ;
 95%  STATE RESET;
 95%  RUNTEST 5 TCK;
 95%  SIR 6 TDI (05) ;
 95%  SDR 160 TDI (0000000400000004800700140000000466aa9955) ;
 95%  SIR 6 TDI (04) ;
 95%  SDR 32 TDI (00000000) TDO (3f5e0d40) MASK (08000000) ;
 95%  STATE RESET;
 95%  RUNTEST 5 TCK;
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```
Restulting in a sucessful flashing of our FPGA. 

I will spare you yet another christmas light demonstration video and will come back to edit this 
article if I spot any issues with the pinnout for the PCIe and SFP+ in the future. 

If we where to take example on the Vivado generated programming sequence in the SVF file, 
we should be able to replicate the programming sequence with openOCD
, allowing us to directly read out the bitstream content and remove the need for a SVF file altogether. 

# Conclusion

For $200 we got a fully working decommissioned Alibaba Cloud accelerator featuring a Kintex UltraScale+ 
FPGA with an easily accessible debugging/programming interface and enough pinout information to define 
our own constraint files.

We also have a fully automated Vivado workflow to implement our designs and the ability to write the bitstream, 
and interface with the FPGA's internal JTAG accessible registers using an open source programming tool without 
the need for an official Xilinx programmer.

In the end, this project delivered a 5x cost savings over commercial boards (compared to the lowest cost $900-1050 Alinx alternatives), 
making this perhaps the most cost effective entry point for a Kintex UltraScale+ board.

# Ressources 

[1] Xilinx Vivado Supported Devices : https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Supported-Devices 

[2] Official Xilinx dev board : https://www.amd.com/en/products/adaptive-socs-and-fpgas/evaluation-boards/ek-u1-kcu116-g.html

[3] Alinx Kintex UltraScale+ dev boards : https://www.en.alinx.com/Product/FPGA-Development-Boards/Kintex-UltraScale-plus.html

UltraScale Architecture Configuration User Guide (UG570) : https://docs.amd.com/r/en-US/ug570-ultrascale-configuration/Device-Resources-and-Configuration-Bitstream-Lengths?section=gyn1703168518425__table_vyh_4hs_szb

