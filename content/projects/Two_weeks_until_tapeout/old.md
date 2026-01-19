---
title: "AI accelerator writeup"
date: 2026-01-05
description: "Taping out a fully featured BLAKE2s hashing accelerator targetting the 130nm SKY130A process."
summary: "BLAKE2s ASIC implementation targetting the SKY130A process, tapedout with Tiny Tapeout."
tags: ["asic", "sky130", "130nm", "rtl", "verilog", "Tiny Tapeout", "DFT", "AI", "Systolic Array"]
draft: true
showTableOfContents: true
---

- Intro 

		

- As anyone that hasn’t been living under a rock might have heard, AI accelerators are the coolest kids in town these days. And although I have never been part of the “in” crowd, this time at least, I get the appeal. 

  So when the opportunity arose to join an experimental shuttle using global foundries 180nm for free I jumped onto the opportunity to submit a systolic MAC accelerator to it. 


  - fun side story as intro : context on the experimental   
    - title:   
      - it seems I never learn my lesson  
      - more terrible ideas   
    - just finished blake2b accelerator \- still recalling from my glorious brunch at WaffleHouse   
    - New bad idea: participate to the experimental shuttle ( “when my brain decided to pitch me its latest masterpiece of poor judgment”; “The setup was perfect for disaster: no project, and a deadline so tight it would barely be enough for you to get all the tools working in a corporate setting” \<- maybe I should keep that one to myself  )   
    - Problem: no plans of what to build, no fixed project, and a very tight deadline  
    - Glarring flaw in the blake2s asic: how to debug if this silicon doesn’t work ?   
    - Focus of the tapeout: DFT with a focus on insilicon observability   
    - Previous projects (among which: link to alibaba cloud accelerator) have given me a good hands on experience with JTAG, but I come from a USER perspective (you know, the easy side where you just complain when things don't work) \-\> opportunity to approach from a designer perspective.   
      - Starting iterating on debug infra fast so I can have proven IP stack to add to future design work   
      - Get a means of inspecting inisilicon so when my tapeout don’t work I have a means of investigating \-\> allowing me to iterate more confidently  
    - But making just a JTAG is a much to reasonable project given the timeline, the silicon area is free, and I am not a reasonable person   
    - Project constraints:   
      - Reasonable area usage: although the area is free I wanted to keep the area reasonable in order to literally give room for other tapeout participants   
      - Somewhat reasonable scope to: designed, tested and emulated in the a reasonable timeline  
      - Something that interests me: I have no shortage of topics of interest \+ I tend to be attracted to projects that push me out of my comfort zone so no: cryptographic/hashing, ethernet, networking, predictors, cpu related projects  
    - So after exploring possibilities (also know as procastinating) for a week, I settled on the extremely original Multiply Accumulate systolic accelerator used for matrix matrix multiplication for AI workloads, not because I am particularly interested in AI but because I am obsessed by how elegant the systolic array architecture are and matrix matrix operations just happen to be a great usercase. Additionally, a matrix matrix operation is easy to validate.   
    - So with less than to weeks to do I set out to designed this accelerator. Under normal circumstances this would have been a terribly tight deadline, and it was, but I deluded myself I thinking I could leverage some of my existing infra from the previous tapeout \+ knowledge  
    - In my pile of terrible ideas this compressed timeline ranks pretty highly \!  
    - This is also why, once again there wasn’t an SRAM: I just didn’t have time

  {{\< figure

  	src=”homo\_electronicus.jpg”

  	caption="\*homo electro-engineerus\* observed in its natural habitat during a pre-tapeout hibernation period known as a 'crunch'. During this period, the specimen will seldom move from its cave, surviving exclusively on a diet of 'coffee' and 'HEB smoked ham'.”

  	alt=”me only my couch”

  \>}}


- Deadline discussion   
  - if this had been a normal corporate environment with this deadline would have been so outrageous people would have laughed at it   
  - but this is an oss project, and the tools where build around the idea of “no human in the loop” and very fast iteration time ( ref openroad design principal ) 

    \> The IDEA program targets no-human-in-loop (NHIL) design, with 24-hour turnaround time and zero loss of power-performance-area (PPA) design quality.

    ref: https://openroad.readthedocs.io/en/latest/\#welcome-to-openroads-documentation

  - librelane follows this principal   
  - the tt flow follows this principal   
  - the entire ASIC flow is optimized for iteration efficiency   
    - reference to my principal of iteration efficiency from the alibaba cloud article   
    - making it possible for the ASIC flow to not be the bottleneck   
  - OSS projects can have some crazy timelines because of how efficient the main flow is   
    - example of spade tt submission   
    - the deadline is now moved to   
        
- Systolic structure \- cost of moving data   
  - power   
  - time  
  - Further memory is the more costly it is   
    - find data, illustrate  
      - cache : registers vs l1 sram vs l2 sram vs l3 vs disk vs network   
      - show time \+ power cost per data transfer   
  - present what systolic means from this perspective   
    - give definition of the systolic array from the paper  
  - present why it is so adapted to AI workloads   
- Present the accelerator’s structure   
  - heart of the accelerator, drop in animated json \<- try to readapt for signed integer rather than floating point

- Booth multiplication   
  - quick overview of different multiplication types   
  - try and find a paper or a ressource presenting different multiplication in terms of area/perf   
  - mention that this accelerator isn’t optimizing for power, so power will not be a metric   
  - delve into booth multiplication   
    - radix explaination   
      - definitaion   
      - translation   
      - radix-4  
    - partial products   
    - sign extension optimization  
      - present sign extension  
      - explain how we can optimize \-\> why it works  
    - signed multiplier : getting ride of last partial product   
      - explain why we can do this   
    - tree addition structure   
    - putting it together   
    - conclusion  
  - mention implementation tools (yosys) can be configured to automatically generate complex multipliers   
    - show config  
    - mention that I didn’t compare the hand rendered booth vs the yosys rendered booth but this this would be interesting  
      - mention that yosys will likely not be able to pick up the signed partial product optimization since verilog doesn’t indicate it is signed ( check this )   
      - parallel with raph’s article on json assembly optimization \-\> sometimes the tools are very hard to beet ( in sw )   
          
- JTAG \- Debug angle  
  - present how the deep data circulating aspect of the systolic array might cause issues for users wanting to debug there application   
  - don’t give an indepth article on the DFT, just mention that it starts to become necessary to help better identify future in Silicon bugs so I am starting to iterate on a DFT intra  
  - give a few words on the current debug features, most importantly the USER REG command   
  - also gives us a very quick quickstart for the chip  

- Conclusion   
  - predictably ended with an all nighter as I sprinted to the tapeout close window   
  - More waffles   
    - caption: Obligatory celebrity waffles, this is becoming a tradition. Some might wonder if an ASIC tapeout might actually be an excuse to go to waffle house. I also share this question.   
  - Each terrible idea lays the groundwork for the next:   
    - this projects builds upon the previousl blake2b asic flow experience, the alibaba cloud openocd jtag mastery  
    - next terrible idea is to iterate on the MAC array but make it support floating point math (because I have found nothing harder than hate writing float math)   
    - full scan chain using openroad dft (for which there is great documentation, it's just a shame it is in  the C++ code) \+ fpga emulation of the scan chain ( link to v1 on github ) 

Phrases to include : 

"The memory hierarchy: proof that in computing, like real estate, it's all about location, location, location."

"Hand-optimized vs tool-generated: the eternal question of 'did I waste my time?'"

As anyone that hasn’t been living under a rock might have heard, AI accelerators are the coolest kids in town these days. And although I have never been part of the “in” crowd, this time at least, I get the appeal. 

So when the opportunity arose to join an experimental shuttle using global foundries 180nm for FREE I jumped onto the opportunity and designed my own JTAG\! 

…

I’m sorry? Is this not what you were expecting? 

Frankly, I would love to tell you a great story about how I went into this wanting to design a free and open source silicon proven AI accelerator that the community could freely extend and re-use in there own projects. But in truth this project started out first as me wanting to design some far less sexy, in-sillicon debug infrastructure and only later came to include the systolic MAC accelerator (AI accelerator) … to serve as the design under test. No wonder I was never one of the cool kids. 

But don’t worry, this title didn’t lie to you, and in this article will focus on the AI accelerator’s architecture, focusing on the why behind the architectural decisions, exploring why systolic architectures are so often used, and the beautiful math allowing you to squeeze even more out of your multiplier optimizations.

So without further udo, let’s jump in \!

{{\< alert “info-circle” \>}}  
I will not be going over the open source ASIC implementation flow in this article, if you are interested in this topic, please refer this the previous article : TODO link   
{{\< /atert \>}}

- Systolic structure \- cost of moving data   
  - power   
  - time  
  - Further memory is the more costly it is   
    - find data, illustrate  
      - cache : registers vs l1 sram vs l2 sram vs l3 vs disk vs network   
      - show time \+ power cost per data transfer   
  - present what systolic means from this perspective   
    - give definition of the systolic array from the paper  
  - present why it is so adapted to AI workloads   
- Present the accelerator’s structure   
  - heart of the accelerator, drop in animated json \<- try to readapt for signed integer rather than floating point  
    

\# Why systolic array rule the seas 

As the heart of many neural network inference operations is the matrix-matrix multiplication, it sometimes feels like every AI accelerator is based on a systolic architecture.   
Here again, when the entire industry starts converging on the same solution is it because there are very valid technical reasons why systolic architectures are particularly well suited out for this workload: from inherent scalability, to its ease of implementation. But I'd like to focus on a different angle, that I believe underlines its greatest strength, but sometimes overlooked: the cost of memory access.

But before unleashing the pJ/bit comparisons, let us take a step back and define the terms of this conversation.

\#\# What is a systolic architecture ? 

\# Fetching data 

DONT KNOW IF I WANT TO INCLUDE : 

reg are D-FlipFlops, very fast, and each single bit storage cell occupies a significant amount of area. For illustrative purposes, for the GF180 PDK this chip is using, here is a weak driven, single bit AND cell compared to a similarly weak driver, simple D-FlipFlop. The AND cell occupied 17.5616 µm2 worth of area and whereas the DFF occupies 65.856 µm2\.   
So reg storage is great until you realise you probably want to actually do some computation with your area and then it becomes a problem of balance.   
{{\< figure   
	src=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_and2\_1.png”  
	caption=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_and2\_1”  
	alt=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_and2\_1”  
\>}} 

{{\< figure  
	src=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_dffnq\_1.png”  
	caption=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_dffnq\_1”  
	alt=”gf180mcu\_fd\_sc\_mcu7t5v0\_\_dffnq\_1”  
\>}}

L1, L2, and L3 are built out of SRAM  
\#\# Power 

Most readers were probably already well aware of the exponentially increasing access times when fetching further, and might have a sense of how this time is probably synonymous with increased energy consumption, but probably lack the understanding of how significantly more expensive in terms of power these memory access are. 

The further data is stored, the more expensive it is to access, this is true both in terms of performance and power. Taking a CPU as an example, accessing data stored in a local register made out of D-Flip-Flops is going to cost me under a cycle and going to be the cheapest access in terms of power, performing a load from L1 will cost me \~3 cycles at best, L2 tens of cycles, L3 tens to hundred of cycles, DRAM hundreds of cycles, and everything afterwards an eternity. 

source : [https://raphaelouthier.github.io/art/doc\_mm\_0\_intro/](https://raphaelouthier.github.io/art/doc_mm_0_intro/)

Here are some numbers for a 64 bit wide access for various memory types, this SRAM’s were calculated for a 45nm node running at 0.9V \[3\]: 

| Memory Type | Energy |  
|-------------|---------|  
| SRAM Cache \- 8KB | 10pJ |  
| SRAM Cache \- 32KB | 20pJ |  
| SRAM Cache \- 1MB | 100pJ |  
| DRAM | 1300\~2600pJ |  
Here we can see how much, if we were to access data we would rather access it from smaller, more local storage. 

But what is really interesting is how this compares to the energy cost of compute : 

| Operation Type | Energy |  
|-------------|---------|  
| Integer \- 8 bit add | 0.03pJ |  
| Integer \- 32 bit add | 0.1pJ |  
| Integer \- 8 bit mul | 0.2pJ |  
| Integer \- 32 bit mul | 3.1pJ |  
| Float \- 16 bit add | 0.4pJ |  
| Float \- 32 bit add | 0.9pJ |  
| Float \- 16 bit mul | 1.1pJ |  
| Float \- 32 bit mul | 3.7pJ |

\> The biggest energy bottleneck in modern chips is in the memory systems. Memory accesses cause energy expensive  
data communication because memories represent global storage whose physical size  
generally grows as technology scales. For example at 40nm technology, the energy of  
64 bit floating point multiply-add is 50pJ while the cost of reading the operand from  
register file is 14pJ \[33\]. In contrast, accessing large cache costs about 1000pJ while  
accessing DRAMs cost about 7000pJ as seen from Table 1.1. Not only is memory  
energy atleast two orders of magnitude larger than computation, it also could scale  
slowly compared to processor. Clearly, unless the energy inefficiency of memory  
systems is addressed, advances in processor will soon be rendered ineffective. 

\> Memory accounts for 25-40% of datacenter energy \[25, 46, 49\] and this percentage will increase as applications de-  
mand larger memory capacities for virtualized multi-cores and memory-based caching  
and storage \[64, 57\]. 

Power estimates for a 45nm process technology : 

| Operations                              | Dynamic power consumption in pJ    |   
|----------------------------------------|-----------|  
| 4x64-bit floating-point multiply-add     | 200 pJ     |  
| 4x64-bit access to 8-Kbyte SRAM\*           | 56 pJ     |  
| 256-bit access to 1-Mbyte SRAM          | 566 pJ    |   
| 256-bit 10mm wire                      | 310 pJ    |   
| 256-bit DRAM interface                 | 5,120 pJ  |   
| 256-bit DRAM access                    | 2,048 pJ  | 

\*Assuming 4 different 8-Kbytes SRAM not a single 8-Kbyte SRAM with 4 64b wide read ports. 

“As we will elaborate later, DRAM access en-  
ergy consumption is largely dominated by the routing inter-  
connect energy consumption that is proportional to DRAM die size”.\[1\]

“Taking video decoding as an example, recent work has shown  
that, an H.264 decoder at 90 nm node only consumes 0.36 pJ per  
pixel, while the corresponding DRAM access energy consump-  
tion is 1.11 pJ per pixel” \[1\]

\#\# Data Re-Use 

One of the great advantages of a systolic array is that data, once inserted into the array, is re-circuled and used in multiple consecutive operations. The more consecutive operations are stacked on a single path, the deeper the systolic network is, the lower the ratio of the cost of fetching data vs compute can be. 

Since our goal is to create the most efficient architecture, we are ultimately looking for the lowest ratio since we will measure our systems performance of what we judge to be a “useful” operation, here the compute. 

Due to how this ratio scales, our incentive is actually in the direction of create the deepest possible network. 

// interactive graph: ratio of power consumption for operation vs graph depth   
// comparison array depth : 1x1, 2x2, 4x4, 8x8, 16x16, 32x32,   
// 8-Kbyte sram vs array depth  
// 

Obviously, there is a limit to this scaling, as we also want to maximize effective workload performance, not theoretical. For example, if I am designing for a custom CNN accelerator, and my system modeling shows me that I would benefit more from 4 8x8 array  

{{\< alert “lightbulb” \>}}  
The matrix–matrix multiplier accelerator was chosen primarily because it is highly amenable to a systolic architecture.  
{{\< /alert \>}}

\[1\] Reducing DRAM Image Data Access Energy Consumption in Video Processing : https://sites.ecse.rpi.edu/\~tzhang/pub/VideoTM2012.pdf?utm\_source=chatgpt.com

					  
						

# **\[2\] The Problem of Power Consumption in Servers** : https://www.infoq.com/articles/power-consumption-servers/

\[3\] Computing’s Energy Problem (and what we can do about it): https://gwern.net/doc/cs/hardware/2014-horowitz-2.pdf?utm\_source=chatgpt.com

