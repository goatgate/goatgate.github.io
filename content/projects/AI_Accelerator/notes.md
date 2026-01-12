title : Homeground AI interfence accelerator



## Design 

This homegrown AI interference accelerator features a tiny systolic array capable of performing a 
matrix matrix multiplication on a 2x2 matrix using signed 8 bit integers. 

It has a maximum operating frequency of 50MHz and an effective max compute capability of 100 MMAC/s (Million Multiply and Accumulate Operations per second)
or 200 MIOP/s (Million Integer Operations per second). 

This most recent ASIC targets the Global Foundries 180nm process GF180, designed using Google's open source GF180-MCU PDK (Process Design Kit) 
with the smaller, more energy efficient but slower 7 segment track standard cells. ( TODO hd cells ? )

Once again, this tapeout is made possible by the Tiny Tapeout program and will be going out as part of the `gf0p2` (GF 0.2) experimental shuttle 
submitted as part of the first wafer.space run. 

{{< alter "heart" >}} 
You can learn more about the awesome Tiny Tapeout shuttle programs at the official website: [https://tinytapeout.com/](https://tinytapeout.com/)

And if you want to go the extra mile and own the full chips check out [`wafer.space` at https://wafer.space/](https://wafer.space/) where you can get 1k ~20mm2 raw dies for as low as 7k USD.
{{< /alter >}}

### Experimental shuttle 

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

{{< figure 
    src="tt_gf02.png"
    caption="Full GDS render of the GF 0.2 Tiny Tapeout chip"
    alt="chip render"
>}}
>
If the final chip is demmed sufficiently functional, the resulting ASICs along with the devnoard will be publically available 
for purchase on the [Tiny Tapeout store](https://store.tinytapeout.com/),


## The Plan

In this article I will be going in depth on the advantages of the systolic architecture from a performance and power efficiency standpoint, optimizing  booth radix-4 for integer multiplications, touching on everyone's favorite topic of building an in silicon debug infrastructure, and of course, the obligatory geeking out on why the OSS design flow is awesome. 

(Because I know everyone is secretly here to read up on in silicon observability and designing JTAG TAPs.) 

The goal of this article isn't to serve as a datashee [(link here for the datasheet)](https://github.com/Essenceia/Systolic_MAC_with_DFT/blob/main/docs/info.md) for this design but rather to discuss what I personally judge to be the more interesting design considerations and optimizations that went into this ASIC.



