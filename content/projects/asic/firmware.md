Requirement : I have very tight timing requirements. 


Seems I am the target demographic : 
```
If you’ve looked at
fixed peripherals on a microcontroller, and thought "I want to add 4 more UARTs", or "I’d like to output DPI video", or
even "I need to communicate with this cursed serial device I found on AliExpress, but no machine has hardware
support", then you will have fun with this chapter.
```
source : 3.1. What is Programmable I/O (PIO)? - Raspberry Pi Pico-series C/C++ SDK 

## DMA 

Each transfert is of 1 to 4 bytes: 
```
Up to 2^32-1 transfers can be performed in one sequence.
```
Joke : since the blake2s hash can accept up to 2^64 bytes this cover all our needs. 

Good for use there is a workaround when using channel chainning : 
```
When a channel completes, it can name a different channel to immediately be triggered. This can be used as a callback
for the second channel to reconfigure and restart the first.
```

--- 

```
2.5.3. Data Request (DREQ)
Peripherals produce or consume data at their own pace. If the DMA simply transferred data as fast as possible, loss or
corruption of data would ensue. DREQs are a communication channel between peripherals and the DMA, which enables
the DMA to pace transfers according to the needs of the peripheral.
The CTRL.TREQ_SEL (transfer request) field selects an external DREQ. It can also be used to select one of the internal
pacing timers, or select no TREQ at all (the transfer proceeds as fast as possible), e.g. for memory-to-memory transfers.
```

We can connect the DMA to the PIO though the DREQ, When the FIFOs will be full or empty, depending on which FIFO 
we are reffereing to, a DREQ will be triggered, waking up the DMA and re-initiating the transfer, 

```
FIFOs also generate data request (DREQ) signals, which allow a system DMA controller to pace its reads/writes based
on the presence of data in an RX FIFO, or space for new data in a TX FIFO. This allows a processor to set up a long
transaction, potentially involving many kilobytes of data, which will proceed with no further processor intervention
```
The TX FIFO sends a DREQ when space is available in the FIFO, not when the FIFO is empty. 

## PIO

You can set none consecutive pins as outputs 
```
(gdb) p/t debug_pio->dbg_padoe
$28 = 10010000001110000000011111111
```

## Ressources 

SDK documentaiton : https://www.raspberrypi.com/documentation/pico-sdk/hardware.html
