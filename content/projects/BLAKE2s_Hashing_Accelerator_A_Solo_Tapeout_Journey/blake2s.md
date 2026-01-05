Blake2s : ASIC implementation 

Into 

- blake2s introduction   
  - presentation   
  - uses  

- Objective of taping this out on tinyTapeout for the sky130A node  
  - solid verification \-\> only one shot   
    - simulation with random stimulus for better coverage of unforeseen corner cases   
    - FPGA emulation for testing alongside co-designed software  ( hot piece of modern art without sw )  
  - actual tapeout \-\> actual manufacturability concerns  
    - fix sky130a drc violations  
    - (just intro ) obey external silicon constraints: IO limitations

ASIC 

- Open source asic workflow overview  
  - Opensoure PDK: SKY130A, foundry provided  
    - implementation targeting the 3v3 operating volate, using high density cells ( \_hd\_ ) variant  
  - Librelane2   
    - short presentation   
    - using classic flow  
  - open source he has never been as accessible: doesn’t mean it is easy

- TinyTapeout shuttle program  
  - Program overview  
  - limitation :   
    - cost scales with area + max area 
    - you can own your own IO but if you use shared IO :   
      - limited on the amount of IO  
      - bandwidth limitations : 66MHz in/ 33MHz out (slow output buffer slew rate due to weak driver) 

ASIC architecture

- Take a step back to talk about blake2  
- different flavors of blake talk about the difference between p, s variants   
  - mention how big the internal state storage is  -> area impact for dff 
  - mention how wide the base data width is \-\> also impacts operation width -> area impact for cominational logic 
  - overview of the area used by dff ( add cells side by side to illustrate point )   

- mention previous blake2b implementation that was heavily optimized for perf running on an FPGA and not connected to internal FPGA logic not external firmeware  
    -> fundamental different archicectures -> entire re-wright 

- External limitations :   
  - low IO bandwidth   
  - max operating freq limited by input IO given bandwidth limitation   
- single tt slot can only store \256b of data if you cover it with dffs, max   

- PPA overview: design trilema   
  - perf \-\> sacrifice power and area   
  - area \-\> sacrific perf   
  - here perf is upper bounded by the IO, this constraints makes this design's direction very clear : 
- Design direction : optimized for area and perf ( up until IO bandwidth limitation )   

     
Design 

- design arch overview   
  - block gather logic over multiple cycles as data comes though 8b wide data interface  
  - once all block data has been acquired start hashing round  
  - if last block, start streaming result hash data out  
 
- Compare to previous blake2b implementation that was perf optimized   
  - b \-\> s  
  - lowered memory usage   
  - previous implem was part of a system, got pull data block over bus, here we need to accumulate block over multiple cycles  
- target internal frequency : 66 MHz
  - broke down G function over 8 cycles 
    - limit access mux for data 
    - access mux are only 4 way mux's  
 - G function is to deep to be executed in a single cycle -> 2 cycles fit 
    + has data dependancies so we can't break operations into independant sub units 
    - incomming data has no dependancy expect on cycle 0 and 4 -> can pipeline without needing to wait for result  
- Slow output mode :   
  - double output data hold cycle mode
    - also applies to hash_valid_o signal 

Firmware co-design 

- RP2040 choice :   
  - used of TT dev board  
  - has PIO allowing high speed parallel bus ( link PIO article )   
- Design modifications to work with sw :   
  - sw starts to capture output data only after valid is asserted : can’t capture data on the same cycle as the valid signal gets asserted like hw   
  - hw sending out data\_ready\_o signal to coordinate when it is ready for more data to be sent ( sw cannot keep accurate track of when hw should be ready )   
  - support for empty data transfers due to dma transfer bubbles when sending data between PR2040’s memory and PIO FIFO TX buffer

Implementation 

- Remind that design was refined alongside timing all this time: we didn’t just decide of final design without running regular implems  
- Overview what max cap, slew rate, antenna violations are  
- Very long area \-\> long wires \-\> multiple such violations  
- How to fix them with the tools  
- How to add buffers into the design to fix last antenna violations that the tools cound not fix  
- reflect on diodes ( placed all over the design in an attempt to fix antenna violations )   
- Overview of final implementation 

Closing thoughts : 

- Thank to TT and opensource community for the tools   
- Possible paths for improvement: wish we had SRAM \-\> re–implement with smaller area footprint ( 26% ( todo check number ) area used for dff ).   
- Add debug intra ( JTAG ) 

Best lines : 

- to describe blake hashing : Imagine a competitive cocktail shaker, but just for bits  
 
- and message lengths up to 2^64-1 bytes (which is roughly "more data than you'll ever actually hash, but we like big numbers").  

- The mission, should I choose to accept it (which I already did, past-tense-me had questionable judgment): tape out a functional Blake2s accelerator on the SKY130A node through Tiny Tapeout.   

- Unlike FPGA development where you can iterate until the heat death of the universe, ASIC tapeout is more like skydiving—you pack your parachute VERY carefully because you only get one shot.  

- The design had to satisfy SKY130A design rule checks, which are basically the **HOA** rules of silicon. "Your metal traces are too close together." "That wire is too long." "Your antenna ratio is unacceptable, we're calling the police." Except instead of passive-aggressive notes, you get manufacturing defects.

- Open-source ASIC design has never been more accessible\! Which is like saying that summiting Everest has never been more accessible because now you can Google "how to climb mountain" and buy crampons on Amazon. Technically true, but you're still gonna have a bad time if you don't know what you're doing.  
    
- Designs using shared I/O face a 66 MHz input limitation. The output? 33 MHz, because the output buffers have the electrical equivalent of low blood pressure. They're trying their best, okay? 

- These constraints would ultimately cap performance more than any internal logic—like having a Ferrari engine but you're stuck in a school zone.

- With limited I/O bandwidth, expensive area, and a frequency ceiling imposed by the input interface, the priorities became crystal clear: minimize area first, extract maximum performance up to the bandwidth limit, and worry about power consumption approximately never.

- This isn’t an arm CPU, I don’t care about your battery life, also half of my flop’s don’t have an enable signal, sue me. 

- You need to store the 512-bit state, the 512-bit message block, and various intermediate values. That's over 1,024 bits of storage just to keep the lights on.

- Design Decisions and Optimizations (Or: Cutting Corners Like A Professional)

- The RP2040 has these beautiful things called PIO (Programmable I/O) blocks that can generate and capture signals with nanosecond-level timing accuracy. This is absolutely critical because **traditional GPIO bit-banging in software is about as precise as throwing darts while blindfolded on a rollercoaster.** PIO state machines can actually keep up with the 66 MHz interface, which is more than can be said for any software running on those ARM Cortex-M0+ cores.

- I made several hardware modifications to accommodate the fact that software is, how do I put this delicately, *“slower than hardware”.* 

- *Not so much. The accelerator asserts a `hash_valid_o` signal one cycle BEFORE data starts flowing, giving the PIO state machine time to go "OH\! OH\! Something's happening\! Deploy the data capture\!" This is the digital equivalent of yelling "INCOMING\!" before throwing something at someone.*

- *The firmware offloads everything time-critical to PIO state machines and high-priority DMA channels. This architecture means the modest ARM cores don't have to keep up with the 66 MHz data rate, which is good, because they absolutely cannot.*

- *Three types of violations kept me company during lonely nights:*

- *682 × 225 µm, which is HUGE by Tiny Tapeout standards—created wires so long they had **frequent flyer miles.***

- It's out there, in fabrication, becoming real silicon. No take-backs now\!

- Currently, we're hoping really hard that everything works. **Hope is not a strategy**, but it's what we've got.

- Hardware without software is just expensive modern art that occasionally gets warm.

- This iterative approach—refine, implement, stare in horror at the results, question life choices, refine again—proved essential. You can't just draw a circuit diagram on a napkin and assume physics will cooperate. Physics is notoriously uncooperative. Physics doesn't care about your dreams. Physics doesn't even return your calls.

- Support for empty transfer cycles (Or: Teaching Hardware To Deal With Awkward Silences)


- Each flip-flop costs area. I started counting them like Scrooge McDuck counting coins, except instead of swimming in gold I'm drowning in standard cells.

- The output runs at 33 MHz because physics said 'no' and I wasn't in a position to argue with fundamental forces of nature.

- At 66 MHz, my ASIC runs at approximately 0.00055% the speed of a modern CPU core. I prefer to think of it as 'retro' rather than 'glacially slow.

- Running DRC is like submitting your homework and having the teacher return it covered in red ink, except the teacher is a computer program with infinite patience for your suffering.

 - Fixing antenna violations is like playing whack-a-mole, except the moles are electromagnetic phenomena and the hammer is more transistors.

- I spent more time thinking about micron-scale area optimization than I spent planning my wedding.

- "Power consumption was so low on my priority list it fell off the bottom and landed in a different project."

- "I cared about power consumption approximately as much as a Formula 1 team cares about fuel economy."

- "The PIO state machines run at nanosecond precision. The ARM cores run at 'eventually' precision."

- "Making chips is now accessible to individuals! Terms and conditions apply. Results may vary. Sanity not included."

- And now I wait until the chip gets back for over 9 months, which is longer than it would take me to make my first child. 

- "The most important skill in ASIC design is learning to make peace with imperfection while obsessing over every detail."

- "My chip is 26 generations behind current technology, which in semiconductor terms makes it basically prehistoric. Like a dinosaur, but smaller and it does cryptography."

- "If you're reading this and thinking 'this sounds hard,' you're correct. If you're thinking 'this sounds fun,' please join the open source silicon community.

- "The memory hierarchy in this design is: registers, more registers, even more registers, and registers.


- "I had 160 × 225 µm to work with. For scale, a human hair is about 75 µm wide. I'm working with roughly two human hairs of area. This explains everything."

- "I ran 10,000 random test vectors. The design passed all of them. I still don't trust it."

- "Verification paranoia: Because the only thing worse than finding a bug in simulation is finding it in silicon nine months from now."

- "Open-source silicon: Where the tools are free, the PDK is free, the knowledge is free, but you'll pay in suffering and existential dread: it's great, 10/10, would recomend.

- "The open-source silicon movement: Proof that humans will do incredibly difficult things just because they can."

- "My chip is fabbed on a process node older than some college students. Don't worry, you are not the only one felling old.. 

- "If you've read this far, you either really care about ASIC design or you're procrastinating. Either way, welcome."
 
- "The best debugging tool is a good night's sleep. The second-best is an ILA with triggers on every suspicious signal."

- "The semiconductor industry: Where we can land robots on Mars but somehow storing 1,600 bits affordably is still considered 'an engineering challenge.'"

- "I had to optimize for area, which is economics-speak for 'you can't afford the good stuff, make it work anyway.'"

- "8 input pins is a generous allocation. Alexander Graham Bell invented the telephone with just TWO wires and he didn't complain.

- "As usual, the testbench has more lines of code than the design itself, which is either thorough engineering or proof that I don't trust my own work. Both are valid."

- "Antenna violations: When your wire becomes a miniature Tesla coil during manufacturing and executes transistors without trial. Due process doesn't exist at the 130nm scale."

- "Is this practical? No. Is it necessary? Absolutely not. Did I do it anyway? Yes. This is called 'engineering for the sake of engineering' and it's a proud tradition."

!!! Check sizes !!!
- "Flip-flops occupy 5.32 µm² each. For comparison, a typical bacterium is about 2 µm². My memory cells are literally larger than life forms. Evolution is more efficient than my RTL."

- "OpenLane is developed by volunteers in their spare time. The commercial EDA tools cost $500,000+ per seat and are developed by companies with billion-dollar R&D budgets. Both produce equivalent levels of frustration. At least one is free."

!!!! Check price difference 
- "Modern chip companies have multi-million dollar NRE costs for tape-out. My NRE is 800 euros. We are both doing chip design. The gap between us is approximately the gap between a second hand toyota corolla and brand new US navy nuclear aircraft carrier. Both are vehicles!"
-














- 
-  
