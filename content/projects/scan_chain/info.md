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
