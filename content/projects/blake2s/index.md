
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


