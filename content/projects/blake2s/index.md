


## My Lattest Bad Idea

Some people have hobbies, I find that cryptographic hashing accelerators are fascinating engineering challenges. 
Please hear me out. Each algorithm offers slightly different design tradeoffs, presents unique opportunities for optimization, 
each serves as a perfect excuse to voluntarily subject yourself to chill firday evening of debugging hash result missmatches.

Now that the stage is set, let me introduce today's star `Blake2`. 
Blake2 is a family of cryptographic hash functions that takes arbitrary amounts of data and compresses it down to a 
variable sized digest (hash), like a unique digital fingerprint used in digital signature algorithms, message authentication codes, and integrity protection mechanisms. 
The Blake2 family comes in two primary variants: 
- Blake2b, designed for 64-bit platforms with its larger internal state and word sizes
- Blake2s, the 32-bit variant optimized for embedded systems and resource constrained environments.

What makes Blake2 particularly interesting is that it was originally designed and optimized for high software performance. 
It's fundamentally a software first algorithm, which makes hardware implementation that much more intrestinglike trying to adapt to hardware. 
I oppose this to an  AES, which maps so clearly to hardware, your archicture practically writes itself as you read the spec. 

### Accessible, Not Easy

The goal of this project was ambitious but clear: independently design, validate, and successfully tape out a fully-featured Blake2s hashing accelerator as an ASIC on the SkyWater 130nm process through the Tiny Tapeout shuttle program.

A few years ago, independent ASIC tape-out would have been a pipe dream unless you arrived with a small New York penthouses worth of disposable income. 

The barriers were: 
- proprietary tools costing hundreds of thousands per seat
- closed-source PDKs requiring NDAs and institutional backing
- fabrication minimums measured in tens to hundreds of thousands of dollars.

Today, thanks to recent advances in open-source EDA tools (Librelane, OpenROAD), the emergence thanks to Google of open-source PDKs (SkyWater 130nm, Global Foundaries 180nm, IHP 130nm), 
and public low cost shuttle programs like Tiny Tapeout that multiplex hundreds of designs onto shared chip, there now exists a path where independent designers can tape out custom ASICs without bankruptcy. 

Some would say ASIC design has never been more accessible.
This statement is technically true in the same way that saying "summiting Everest has never been more accessible" 
is true because you can now Google "how to climb mountain" and buy crampons on Amazon. 
But accessible doesn't mean easy. The mountain still needs climbing, the oxygen still gets thin at altitude, and you can still die of exposure if you make poor decisions.

But accessible it is, and that's revolutionary.

### One Shot, Don't Miss

Given this was an actual tape-out with real fabrication costs ($800+ per run), significant time investment, and a nine-month turnaround from submission to receiving silicon, 
verification wasn't just an after though it was a central driving force. 

Unlike FPGA development and software where patches are possible, ASIC bugs are permanent, expensive, and occasionally end up as cautionary tales in textbooks.

As such, this verification strategy employed a multi-layered defensive line:
- **Simulation-based verification** formed the foundation. The design results where simulated using cocotb against an instrumented golden model (hashlib's blake2s version). 
Testing included both directed test cases for the official test vectos and constrained random stimulus generation for broader coverag. 
The goal wasn't just functional correctness, it was finding the bugs that an overly directed approach testing would miss.

.
- **FPGA emulation** because hardware without software is just expensive modern art that occasionally gets warm. Emulation provided the critical bridge between simulation and silicon. 
The design was ported to a Basys3 FPGA (Xilinx Artix-7, board chosen for it's abundance of IO pins and next day amazone shipping) and connected to an RP2040 microcontroller via GPIO, 
recreating the hardware-firmware interface planned for the final ASIC. 
This environment enabled co-design and validation of both the accelerator and its firmware, catching protocol issues, and integration bugs that only manifest in real hardware operating at full speed.


## Manufacturing: Where Theory Meets Reality

On the ASIC side, correctness and timing were necessary but not sufficient. An actual tape-out introduces the additional constraint of manufacturability—ensuring the design can be physically fabricated with acceptable yield.

This involved two major challenges:

**Design Rule Checking (DRC)** for the SKY130A process meant satisfying thousands of geometric constraints: minimum metal widths, spacing rules, via enclosures, antenna ratios, and metal density requirements. Violations aren't suggestions—they're paths to manufacturing defects or outright fabrication failure.

**Shuttle-imposed limitations** required designing around constraints inherent to the Tiny Tapeout platform itself. The shared I/O architecture imposed severe bandwidth limitations: 66 MHz input maximum, 33 MHz output due to weak buffer drivers, and only 24 total pins for all communication. These weren't just inconveniences—they fundamentally shaped the architecture, capping performance more than any internal logic constraints.

## What Follows

This article chronicles the journey from concept to tape-out: the architectural decisions driven by area constraints, the optimization strategies necessary to fit Blake2s into a footprint measured in hundreds of microns, the physical implementation challenges of routing 12,847 standard cells while satisfying Manhattan-style routing rules, and the firmware co-design required to actually use the accelerator.

It's a story of tradeoffs—performance versus area, elegance versus pragmatism, best practices versus available resources. It's also a story of what's now possible for individual designers in the era of open-source silicon.

The chip is currently in fabrication. In three months, we'll know if six months of work produced a functional cryptographic accelerator or an expensive lesson in what not to do.

Either way, it's been one hell of a climb.


