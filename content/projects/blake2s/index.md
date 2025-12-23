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
current blocks, 64-128 Bytes (b)
internal state used by the mixing function 64-128 Bytes (v)
current block result 32-64 Bytes (h)

As a reminder, a single tile can fit 440 bits or 55 Bytes worth of D-FlipFlops at the very maximum, if you optimize only for dff count and push routing to the absolute limit. This is because D-FlipFlop cells, a combination of two latches, are some of the largest standard cells in the library. 

For illustrating this area scale difference, here is the `sky130_fd_sc_hd__and2_1` cell, a 2 input and gate cell with a weak driver: 

And here is the `sky130_fd_sc_hd__dfxbp_1` cell, a standard, complementary output d-flip-flop with the same weak output drive strength. 


This flip-flop occupies 4 times the area of a simple and gate. 

Yet, here we are looking to store at the strict and very optimistic minimum 3 to 6 tiles worth of just flip-flops just to get this project off the ground. 

But, when it comes to on chip storage I have two major issues : 
There is currently no proven open source SRAM for sky130. Though an experimental SRAM macro was submitted alongside this design on this shuttle including, given it is unproven, it would have risked losing the chip due to a bug. 
D-Flipflop cells, now my primary source of storage, each storage element takes up a lot of area. 
In a perfect universe, given the relatively massive amount of storage and the access pattern, storing data to an SRAM would have been ideal. In practice, using flops was my only feasible path. 

Given that the blake2b variant requires twice the on chip storage compared to the blake2s version, this area constraint guided the choice to implement the blake2s variant. 

#### I/O bottleneck 

The objective was to tapeout this design as part of the sky2b tinytapeout shuttle, as such, this design will be integrated as a pre-hardened macro block into the larger chip. Like most blocks, it will communicate with the pins though a shared mux, and not own any pins on its own. 
In addition to a reset and clk signal, each block is given access to the following I/O : 
8 input pins
8 output pins
8 configurable input/output pins

> Although there is an option to purchase extra design exclusive pins, these cost an extra 100 euros per pin and are limited in number. 

Since these are the shuttle’s shared I/O such, the design must respect whatever operating characteristics this shared I/O access imposes.  And, this is where two additional limitations arise:

Firstly, the gpio cell has a characterized maximum operating frequency of 66 MHz when operating above 2.2V (we are operating at 3.3V). In practice, this caps the maximum frequency at which we can operate the parallel bus between our MCU and our design going over these pins. 

Secondly, because of a weak driver on the output path (Design to MCU) leading to a slow slew rate, the maximum reliable operating frequency is 33MHz. Meaning that any transitions sent out over the parallel bus must not have a transition frequency over 33MHz or else risk signals being captured at the wrong level. 


##### Putting it together 

Given that additional metadata must be transmitted alongside the input data, and putting aside a few more bits for handshaking with the MCU, we are left with only 8 bits in both the input and output direction for data transfer. Given that, each hashing round of the mixing function is performed on an entire blocks data, before and compute can start, we must first accumulate all 64 Bytes of the next block before it can start computation. 

Additionally, due to our area limitation, we can’t afford to spend an extra 64 Bytes, and pipeline this accumulation in parallel with the previous block’s compute. 
As such, compute is stalled while we accumulate the next block. 

In order the comply with the GPIO operating frequency limit, could operate the accelerator at 33 MHz and align the input bus frequency with the output frequency. But, because of the bottleneck imposed by the significant amount of cycles required to transfer the next block data, having an input bus with the highest operating frequency possible was paramount. As such, I made the decision to operate both the input and output bus at 66MHz, and compensate for the slow slew rate by holding data on the output interface for double the cycles. 
Although it would have technically been possible to introduce a new faster internally internal clock domain to the accelerator and simply add a clock domain crossing between the internal logic and the input/output interfaces, given this internal clock domain would have been used to accelerate compute, and this design is bottlenecked by waiting on the input block data, doing so would not have yielded much of an improvement. 


### Design 

Given all these external constraints, the design’s direction was clear: this would be a blake2s implementation, with on chip storage and a focus on optimizing for area and a target operating frequency of 66 MHz.

#### I/O bus protocol 

Given have 8 input, 8 output, and 8 configurable input/ouput pins at our disposal to communicate between the MCU and the accelerator, the goal is to design a parallel bus protocol capable of  

#### Hash configuration 

During each hash function run the internal state (h) either during initialization of the first block, or during initialisation on the last block, the value of the internal variable is calculated based on a set of per hash function run configuration parameters. 
These values are : 
kk, key length in bytes
nn, resulting hash length in bytes
ll, total length of the data to be hashed
In the software implementation, these are passed to the blake2s function when it is initially called. For our design given we cannot derive these from the input data, we also need a way to acquire these configuration parameters before the hash computation begins. 

As such, in parallel to the block data transfer packet exists a configuration parameter transfer packet. This packet is set over the same 8 bit data input interface and is differentiated from the data transfers by the state of the data mode pins. 

The packet has the following layout, uses little endian, and when sending multiple-byte-long arrays, the lower indexes are sent first : 


The `byte_size_config` module (that lives under the `io_intf` module) identifies these configuration packets, parsing of this packet occurs. It output the latest seen values of `kk`, `nn` and `ll` directly to the main hashing modules, as such, the same configuration can be re-used across multiple runs of the accelerator. 

#### Block data buffering

A block data needs to be streamed over 64 cycles from the MCU to the ASIC. Like with the config parameter, there is a module dedicated in the design to tracking this data arrival, but because of the size of the buffer, directly translating to the area occupied by dffs, needed to hold this data. The block isn’t stored in memory in this module and it’s content is held stable over an interface in destination to the main hashing module. Rather, this module contains only the logic needed to identify when data is being received, the data offset, and a copy of the most recent byte. On the main hashing module side, this signals are received, and used to determinin the appending on the next incoming data byte to the block buffer. 

This split allows there to be only the strictly necessary logic dedicated to this block data reception logic in the main hashing module while putting the majority of the block data streaming logic in the `block_data` also under the `io_intf`. 

Now earlier I said that a byte index was included in the signals sent from the `block_data` to the main hashing module `blake2s`. 
Contrarily to what intuition might suggest this byte index isn’t used to address the buffer on writes, the reason as to why is that this would require a 1-to-64 x8 wide demux logic to implement. This would be quite expensive in terms of area. Rather, we are using a less expensive shift buffer to write and works well for our application given bytes are streamed in order to the ASIC. 

> If readers are interested in learning more about the area impact of different memory storage structures for sky130 `Toivoh` did a great study that can be found : [https://github.com/toivoh/tt08-on-chip-memory-test](https://github.com/toivoh/tt08-on-chip-memory-test)

This byte index is actually used to keep track of the completion of the block data so that the main hash module can start computing the hash for the next block. 

#### Mixing function 

The mixing function is at the heart of the hashing function. 

todo

#### Output streaming 
 
At the end of the hash computation of the last data block the ASIC starts streaming out the hash result. Eventhough 32 final hash bytes have been computed, only the first `nn` bytes will be streamed out ( dependent on the `nn` configuration parameter). Due to our slow slew rate on the output GPIO buffer, a “slow output mode” was designed who’s role is to hold each data transfer out of the ASIC over 2 cycles rather than one, leaving enough time for the signal to settle on the output pins. 

The output streaming logic lives in the main hash module `blake2s` but result streaming doesn’t block the fsm from starting to wait for the next start of block to stream in. 



