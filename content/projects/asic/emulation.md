# Pmod pin mapping 

Using JA, JB and JC


Listing IO BANK : 
```
Vivado% foreach p [get_package_pins -of_objects [get_ports -regexp pmod.* ]] {puts "[get_property NAME $p]:[get_property BANK $p]"}
G3:35
H2:35
K2:35
H1:35
G2:35
J2:35
L2:35
J1:35
C16:16
C15:16
A17:16
A15:16
B16:16
B15:16
A16:16
A14:16
R18:14
P17:14
M19:14
L17:14
P18:14
N17:14
M18:14
K17:14
```

List of clk capable pins : 
```
Vivado% get_package_pins * -filter {IS_CLK_CAPABLE == 1}
M18 M19 L17 K17 N17 P17 P18 R18 C15 B15 A16 A17 C16 B16 C17 B17 U4 V4 W5 W4 W7 W6 U8 V8 N3 P3
```


# JTAG

# detecting digilient board
start hw manager and connect 
```
open_hw_manager
connect_hw_server
```

probe for available xilinx approved jtags : 
```
Vivado% get_hw_targets
localhost:3121/xilinx_tcf/Digilent/210183BE6AFBA
```
just works TM

# Ressources 

Basys3 reference : https://digilent.com/reference/programmable-logic/basys-3/reference-manual?redirect=1
