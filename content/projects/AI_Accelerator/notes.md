title : Homeground AI interfence accelerator

## Design 

This homegrown AI interference accelerator features a tiny systollic array capable of performing a 
matrix matrix multiplication on a 2x2 matrix using signed 8 bit integers. 

It has a maximum operating frequency of 50MHz and and effective max compute capability of 100 MMAC/s (Million Multiply and Accumulate Operations per second)
or 200 MIOP/s (Million Integer Operations per second). 

This most recent ASIC targets the Global Foundery 180nm process GF180 and was designed with Google's open source GF180-MCU PDK (Process Design Kit) 
using the smaller, more engergy efficient but slower 7 segment track standard cells. ( TODO hd cells ? )

Once again, this tapeout is made possible by the Tiny Tapeout program and will be going out as part of the `gf02` experimental shuttle 
submitted as part of the first waffer.space run. 

Tiny Tapeout experimental shuttles are used as testing grounds for fabs and processes, they are used to iron out
and issues before opening the classic shuttles. Participation to these experimental shuttles is generally reserved to people having
previously participated in a Tiny Tapeout tapeout, thus, my hashing accelerator submitted as part of the [`sky25b` shuttle](TODO)
allowed me to participate. 

This limitation was imposed to help select for more experienced designers as these experimental shuttles 
are used as a the testing ground for new design flows and as the final chip doesn't feature all the inter design issolation as the 
classic flow. \
For example, in the classic flow, each submitted design macro was individually power gated, and only
powered when selectively enabled. In the experimental shuttle, each designer is trusted to gate there own design, 
if not power or clock gated, at least to reduce dynamic power consumption when not selected. \
As another example, in classic flows, the maximum I/O opperating characteristics are well understood (eg: for SKY130 we have 66 MHz input, 33 MHz output and 2 ns maximum inter-pin skew),
in the experimental flow we don't yet have certainty as for how far we can push the I/O. 
In my design, for the AI acclerator part, I assumed both the input and output paths could at least reach 50 MHz, but 
it is quite possible this assumption might turn out to be too optimistic. 

As such, these experimental shuttle chips have many additional risk that the chip itself, and not 
individual designs, might not be fully functional. \
Knowing this limitation, the Tiny Tapeout program is very generously making submissions to these shuttles free of charge. 
In parctice, this means that area is effectively free explaining the higher ratio of very 
large designs being submitted. 

If the final chip is demmed sufficiently functional, the resulting ASICs along with the devnoard will be publically available 
for purchase on the [Tiny Tapeout store](TODO link),

 going out as part of a TinyTapeout shuttle <3
This homegrown AI inference accelerator features a 2x2 systolic matrix multiplier on GF180 supporting multiply and accumulate operations on int8 data alongside a design for test infrastructure to help debug both usage and diagnose design issues in silicon.

This MAC accelerator operates at up to 50 MHz and is capable of reaching up to 100 MMAC/s or 200 MIOPS/s.


## The Plan

Like allways, the plan for this article isn't to be a datasheet for this accelerator (you can find the [datasheet here](TODO github link)), but 
to present the design principals and some of the more intresting opptimization that when into this homeground AI inference accelerator. 


{{< alert "info-cirle" >}}
Accelerator data
{{< /alter >}}
