## My Latest Bad Idea

Some people have hobbies, I find that cryptographic hashing accelerators are fascinating engineering challenges. 
Please hear me out. Each algorithm offers slightly different design tradeoffs, presents unique opportunities for optimization, easy to validate and each serves as a perfect excuse to voluntarily subject yourself to a Friday evening of debugging hash result missmatches.

Now that the stage is set, let me introduce today's star `Blake2`. 
Blake2 is a family of cryptographic hash functions that takes arbitrary amounts of data and compresses it down to a 
variable sized digest (hash). This hash is then used as a digital signature for message authentication codes, and integrity protection mechanisms. 

The Blake2 family comes in two primary variants: 
- Blake2b, designed for 64-bit platforms
- Blake2s, the 32-bit variant

What makes Blake2 particularly interesting is that it was originally designed and optimized for high software performance. 
It's fundamentally a software first algorithm, in contrast to an  AES, which maps so clearly to hardware, your architecture practically writes itself as you read the spec. 

### Accessible, Not Easy

The goal of this project was to independently tapeout my first ASIC outside of any organized team. This involves solo: designing, validating, and successfully tape out a fully-featured Blake2s hashing accelerator on the SkyWater 130nm process through the Tiny Tapeout shuttle program.

A few years ago, independent ASIC tape-out would have been a pipe dream unless you had successfully sold your startup to meta or grew up in a family whose garage included a private jet. 

Today, thanks to recent advances in open-source EDA tools (Librelane, OpenROAD), the emergence thanks to Google of open-source PDKs (SkyWater 130nm, Global Foundaries 180nm, IHP 130nm), 
and public low cost shuttle programs like Tiny Tapeout that multiplex hundreds of designs onto shared chips, there now exists a path where independent designers can tape out custom ASICs without bankruptcy. 

Some would say ASIC design has never been more accessible.
This statement is technically true in the same way that saying 
running a marathon has never been as accessible: you can get a decent pair of running shoes from your local sports store, and running outside is free. 

But accessible it is, and that's revolutionary.

### One Shot, Don't Miss


Unlike FPGA development and software where patches are possible, ASIC bugs are permanent, expensive, and occasionally end up as cautionary tales in textbooks.

As such, verification became my obsession. This verification strategy employed a multi-layered defensive line:
- **Simulation-based verification** formed the foundation. The design results were simulated using cocotb against an instrumented golden model (hashlib's blake2s version). 
Testing included both directed test cases for the official test vectors and constrained random stimulus generation for broader coverage. 
The goal wasn't just functional correctness, it was finding the bugs that a directed approach testing would miss.
- **FPGA emulation** at the top, because hardware without software is just expensive modern art that occasionally gets warm. Emulation provided the critical bridge between simulation and silicon. 
The design was ported to a Basys3 FPGA (Xilinx Artix-7, board chosen for it's abundance of IO pins and next day shipping on amazon) and connected to an RP2040 microcontroller via GPIO, 
recreating the hardware/firmware interface planned for the final ASIC. 
This environment enabled co-design and validation of both the accelerator and its firmware, catching protocol issues, and integration bugs that only manifest in the real hardware/MCU (MicroController Unit) setup.


### Theory Meets Reality

On the ASIC side, design correctness and timing passing were necessary but not sufficient. 
An actual tape-out introduces the additional constraint of: can we actually build this, can we get this to fit within the characterized operating parameters of the standard cells. Aka: if we do build it, will it work ? 

This challenge involves:
- **Design Rule Checking (DRC)** for the SKY130A process this meant staying within acceptable antenna ratios, and keeping all the wire slew rates and max capacitances within PDK spec. 
- **Shuttle-imposed limitations** required designing within the constraints inherent to the Tiny Tapeout shuttle chip. The shared I/O architecture imposed 
severe bandwidth limitations: 66 MHz input maximum, 33 MHz output due to weak buffer drivers, and only 24 total pins for all communication. 
These weren't just inconveniences, they fundamentally shaped the architecture, capping performance more than any internal logic constraints.

### What comes next

This article goes through the process from idea to tapeout. 

The chip is currently in fabrication and in nine months, we'll know if this produced a functional cryptographic accelerator or an expensive paperweight.


























## Open Source Silicon 

Before jumping deep into the weeds, I would first like to take a step back and give some context about the tools and shuttle program I will be using for this tapeout. 

This won’t be an exhaustive list, and there are plenty of amazingly powerful tools that I won’t be mentioning, think of it as a list of my favorites. 


> If you are already familiar with the Open Source Silicon ecosystem and workflow you can skip this section. 

### OpenRoad 

OpenRoad isn’t the name of a single tool as much as it is a family of ASIC design tools and is at the heart of the ASIC flow. 

In conjunction with OpenSTA, which contains the timing engine, it includes the tools needed from taking a design from floorplanning through placement, clock tree synthesis, routing all the way to finishing. 

The official OpenRoad documentation has a great breakdown of some of the main features : 

Here are the main steps for a physical design implementation
using OpenROAD;

- `Floorplanning`
  - Floorplan initialization - define the chip area, utilization
  - IO pin placement (for designs without pads)
  - Tap cell and well tie insertion
  - PDN- power distribution network creation
- `Global Placement` 
  - Macro placement (RAMs, embedded macros)
  - Standard cell placement
  - Automatic placement optimization and repair for max slew,
    max capacitance, and max fanout violations and long wires
- `Detailed Placement`
  - Legalize placement - align to grid, adhere to design rules
  - Incremental timing analysis for early estimates
- `Clock Tree Synthesis` 
  - Insert buffers and resize for high fanout nets
- `Optimize setup/hold timing`
- `Global Routing`
  - Antenna repair
  - Create routing guides
- `Detailed Routing`
  - Legalize routes, DRC-correct routing to meet timing, power
    constraints
- `Chip Finishing`
  - Parasitic extraction using OpenRCX
  - Final timing verification
  - Final physical verification
  - Dummy metal fill for manufacturability
  - Use KLayout or Magic using generated GDS for DRC signoff

It was originally funded by DARPA and this thought warms my aching heart when I see my tax bill. 


### LibreLane 

Librelane is like a makefile the ties all of the ASIC tools together to build the ASIC flow. 

It pulls in Verilator for lint checking, Yosys and abc for synthesis, OpenROAD for implementation, augmented with a few additional custom scripts to help complete the flow. 

For this tapeout we will be using the “classic” variant of the librelane flow. 

### Sky130A PDK 

The PDK for Product Design Kit, is an integral part to making a digital ASIC design as it contains the cell libraries and their characterizations for target operating parameters (temperature/voltage). It is literally the foundation upon which a digital design is built. Because of their nature PDK’s are foundry process specific, each PDK is tailored to the nature of a foundry and is typically designed by or in close cooperation with said foundry. 

Typically getting access to a PDK requires at least signing an NDA with a foundry, and isn’t accessible to just anyone, so when Google partnered with the US based Skywater fab to release an open source PDK for there 130nm process. Google has additionally partnered with global foundries and released the gf180 for there 180nm MCU process. These open source PDKs have truly changed what was possible for open source silicon. 

For this tapeout I will be targeting a sky130A process, this is a classic CMOS process without the magnetic RAM that differentiates it from the sky130B process. 
More specifically for this tapeout I will be targeting the high density cell library `hd` and a typical operating temperature of 25C for a voltage of 3.3V. 

### TinyTapeout

TinyTapeout is an open shuttle program where multiple projects are pulled together and tapped out as part of the same chip. Participant projects are hardened as macro blocks, the size of which scales with the amount of purchased tiles. As such, projects range from the very tiny counter, to the more larger SoC. These projects are multiplexed together behind a shared mux, connected to a shared bus, and use common I/O (with a few exceptions). 
The final chip users can program which project they want to enable. 

The final chip is sold as part of a dev board which contains both the tinyTapeout chip, and is connected to a RP2040 MCU. 



## Architecture 

As I said in the introduction, something that I find makes hashing algorithms particularly interesting to implement is how the same functionality can be designed for so differently given different PPA (power, performance, area) targets. 
And in this project, there were a few external constraints that heavily weighed on this architectural direction. 

### Realities constraints

#### Area 

A single sky130 Tiny Tapeout project tile, is, as the name forshadows, tiny … About 161x111.52 um or, about as large as to fit 256 bits worth of flip-flops on a good day.

Projects come in a set of varying sizes, from the smallest and more common single tile project, all the way to the 24 tile behemoth. But size isn't the only thing that scales, cost too scales, with a single tile costing 70 euros, projects thus range from 70 to 2240 euros. 

I might be a little cheaper than the next person, so I will fight hard to save myself from parting with a few extra hundred euros.
Unfortunately, I had not chosen the most favorable project in the circumstance. 

##### Blake2

In order to better explain why a blake2 implementation is such a challenge from an area optimization perspective, we must first take a step back and look at the algorithm itself. 

As stated in the introduction, blake2 is a hashing function, its objective is to take a large amount of data and produce a high entropy and reproducible shorter hash. This hash can then be compared against an expected hash result to identify if the data has been corrupted or tampered with. 

Internally, the blake2 hashing function breaks down the incoming data to be hashed into blocks of 64 bytes (blake2s) or 128 bytes (blake2b) and makes each blocks go though 10 (blake2s) or 12 (blake2b) rounds of a mixing function. This mixing function is performed on a per-block internal state the size of which matches the block size. The block result is then derived from this internal state and the final hash result is derived from the final block’s final result. 

From this explanation we can clearly see not only how blake2b needs twice the storage requirement of blake2s but also how large the memory footprint of blake2 is. 

##### Storage

So at a minimum, I need to store : 
current blocks, 64-128 Bytes
internal state used by the mixing function 64-128 Bytes (v)
current block result 32-64 Bytes (h)

As a reminder, a single tile can fit 256 bits or 32 Bytes worth of D-FlipFlops if routing likes you, and here we are looking to store between 5 to 10 times just to get this project off the ground. 

But, when it comes to on chip storage I have two major issues : 
I currently have no knowledge of a proven SRAM for sky130. Though an experimental SRAM macro was submitted alongside this design on this shuttle including, given it is unproven, it would have risked losing the chip due to a bug. 
D-Flipflop cells, now my primary source of storage, each storage element takes up a lot of area. 
In a perfect universe, given the massive amount of storage and the access pattern, storing data to an SRAM would have been ideal. In practice, using flops was my only feasible path. 

Originally I was planning on implementing the more popular blake2b, but after initial implementation runs showed this might have required a 24 block project (2240 euros) the design was re-focused on the smaller blake2s variant.  

