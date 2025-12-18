


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
- **FPGA emulation** because hardware without software is just expensive modern art that occasionally gets warm. Emulation provided the critical bridge between simulation and silicon. 
The design was ported to a Basys3 FPGA (Xilinx Artix-7, board chosen for it's abundance of IO pins and next day shipping on amazone) and connected to an RP2040 microcontroller via GPIO, 
recreating the hardware/firmware interface planned for the final ASIC. 
This environment enabled co-design and validation of both the accelerator and its firmware, catching protocol issues, and integration bugs that only manifest in the real hardware/MCU (MicroController Unit) setup.


### Theory Meets Reality

On the ASIC side, design correctness and timing passing were necessary but not sufficient. 
An actual tape-out introduces the additional constraint of: can we actually build this, can we get this to fit within the caracterized operating parameters, if we do build it, will it work ? 
manufacturability—ensuring the design can be physically fabricated with acceptable yield.

This challenge involves two major axis:
- **Design Rule Checking (DRC)** for the SKY130A process this meant staying within acceptable antenna ratios, and keeping all the wire slew rates and max capacitances within PDK spec. 
Violations aren't simple suggestions, antenna violations for example translate directly, depending on how bad the violations are and there number, to manufacturing defects resulting in lower yield.
- **Shuttle-imposed limitations** required designing within the constraints inherent to the Tiny Tapeout shuttle chip. The shared I/O architecture imposed 
severe bandwidth limitations: 66 MHz input maximum, 33 MHz output due to weak buffer drivers, and only 24 total pins for all communication. 
These weren't just inconveniences, they fundamentally shaped the architecture, capping performance more than any internal logic constraints.

### What comes next

This article chronicles the journey from concept to tape-out of an the architectural decisions driven by area and bandwidth constraints.
It's also a tale of what's now possible for individual designers in the era of open-source silicon.

The chip is currently in fabrication and in nine months, we'll know if this produced a functional cryptographic accelerator or an expensive paperweight.



