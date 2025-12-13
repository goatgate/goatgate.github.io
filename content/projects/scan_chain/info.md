Openroad doesn't keep track of scan chain configurations in the ODB. 

Once a scan chain has been backed in, nothing tells you a scan chain has been inserted. 

One way of checking available scan chains is though the `def` files. 

```tcl
set def_file [make_result_file tes.out.def]
write_def $def_file
```

To know if a scan chain is present, check for the presence of the `SCANCHAINS` lable, eg :  

```
SCANCHAINS 1 ;
```


## Openroad 

Get name of net connected to Q on an sdff instance 
```tcl
openroad> get_name [[$inst find_pin Q] net]
m_bsc_result_out.g_bsp_inner[3].m_inner.ff_2_q
```

Get instance of cell by name:
```
openroad> get_cells _4170_
_40c85b0fc4550000_p_Instance
```



## Dont touch commint back to bite me 

When trying to automatically connect 
```
WARNING: [Netlist 29-48] Cannot modify dont-touch net 'm_top/m_bsc_result_out/g_bsp_inner[5].m_inner/ff_2_q'.
ERROR: [Coretcl 2-1653] could not connect net 'm_top/m_bsc_result_out/g_bsp_inner[5].m_inner/ff_2_q' to requested pins.
ERROR: [Common 17-39] 'connect_net' failed due to earlier errors.
```

## Not done yet 

After believing the insertion was sucessfull I am greeted to this opt_design failture ... so sad
```
opt_design failed
ERROR: [Opt 31-67] Problem: A LUT3 cell in the design is missing a connection on input pin I0, which is used by the LUT equation. This pin has either been left unconnected in the design or the connection was removed due to the trimming of unused logic. The LUT cell name is: m_top/m_bsc_result_out/g_bsp_inner[4].m_inner/ff_1_q_reg_scanmux/res_o_INST_0.
```


## Getting the scan chain

As the scan chain doesn't apprear in the original HDL code, and the order of the scan flops in the 
chain is implementation defined, we must extract the scan chain topology from the implementation results. 

## Renamping to flops

We obtained the chain ordered by scan dff cells, the next step is to remap these abstract cell names to the 





