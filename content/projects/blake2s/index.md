

## Introduction 

birth of a bad idea. 

Cryptographic hashing accelerators are intresting, each one offers a slightly different design challenge, 
and give way to 


Blake2 is a family of cryptographic hashing function, it takes a large amount of data and hashes it down 
to a unique few bytes of data, it is used in  digital signature algorithms, 
message authentication and integrity protection mechanisms. 
Blake2 was originaly designed and optimized for high software performance, meaning it is a a software first algorythm, 
making it's implementation in hardware that much more intresting. 
There are two main flavors of Blake2 : 
- blake2s and blake2b, blake2b was designed for 64 bit platforms and blake2s for 32 bit platforms. 

The objective of this project was to design a fully featured blake2s hashing acclerator from scratch and 
succeffuly tape it out as an ASIC. 

A few years ago tapeing out an ASIC independantly would have been a pipe dream unless I came prepared with 
a few new york studios worth of cash to spare. 
Today, thanks for recent leaps in open source eda tools, the existence of a widdening selection of open source PDKs, and 
open shuttle programs like tinyTapeout, there now exists a path where independants can now tapeout there own ASIC without brankrupting themselves. 
Some would say ASIC tapeout has never been more accessible, this is true. 
But this is also like saying that summiting Everest has never been more accessible because now you can Google "how to climb mountain" and buy crampons on Amazon. 
Accessible doesn't mean easy, you still need to climb the mountatin.

This post is a recollection of my first asic tapeout. 

The objective of this project was to independantly design, validate and tapeout a blake2s hashing accelerator 
as part of the tinytapout shuttle using the Skywater 130nm A process.

Given this was an actualy tapeout, that they are still rather pricey, and that they have very long turnarounds, 
verification was very important for this design.
First, this design was validated using a simulated testbench, using both directed and then random stimulus 
for better coverage. 
Then, since my ASIC without software would be nothing more than an expensive piece of modern art that would 
occasionally get warm, this design was ported onto an FPGA for emulation, and the firmware was co-designed 
alongside the accelerator, allowing testing for the final configuration where the accelerator would communicate
with an MCU. 

On the ASIC side, since we are doing an actual tapeout, on top of all the usual corectness and timing issues, 
we not had the additional constraint of manifacturability to consider for this design. 
This imvolved fixing all the DRC violations for the sk130a process but also, designing around the limitations
on out design imposed by the shuttle chip itself, particularly the IO bandwidth limitation. 





