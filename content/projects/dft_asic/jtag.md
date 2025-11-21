
## Mandatory JTAG instructions 

The four IEEE 1149.1 defined mandatory JTAG instructions IDCODE, BYPASS, SAMPLE/PRELOAD, and EXTEST.

## Boundary Scan 

The EXTEST instruction is used for sampling external pins and loading output pins with data. 
The data from the output latch will be driven out on the pins as soon as the EXTEST instruction 
is loaded into the JTAG IR-register. Therefore, the SAMPLE/PRELOAD should also be used for setting 
initial values to the scan ring, to avoid damaging the board when issuing the EXTEST instruction 
for the first time. SAMPLE/PRELOAD can also be used for taking a snapshot of the external pins 
during normal operation of the part.

source : https://onlinedocs.microchip.com/oxy/GUID-74F8229E-4C43-4FA0-BE7D-1AA303C6F8A4-en-US-6/GUID-86F1AB9A-120D-42D4-8B59-7A9F04E34236.html?hl=extest


### EXTEST 

The active states are:

- **Capture-DR**: Data on the external pins are sampled into the Boundary-scan Chain.
- **Shift-DR**: The Internal Scan Chain is shifted by the TCK input.
- **Update-DR**: Data from the scan chain is applied to output pins.

## Ressources 


AVR JTAG System Overview : https://onlinedocs.microchip.com/oxy/GUID-74F8229E-4C43-4FA0-BE7D-1AA303C6F8A4-en-US-6/GUID-86F1AB9A-120D-42D4-8B59-7A9F04E34236.html?hl=extest

