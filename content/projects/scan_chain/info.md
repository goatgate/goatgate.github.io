---
draft: true
---

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

## Getting the scan chain

As the scan chain doesn't apprear in the original HDL code, and the order of the scan flops in the 
chain is implementation defined, we must extract the scan chain topology from the implementation results. 

## Renamping to flops

We obtained the chain ordered by scan dff cells, the next step is to remap these abstract cell names to the 





