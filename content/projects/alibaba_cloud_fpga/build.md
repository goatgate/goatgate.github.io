
### Creating the project 

// creating the project 

Our first step is going to be creating a new vivado project for our target part `xcku3p-ffvb676-2-e` using the `create_project` tcl command,  
```
create_project -part xcku3p-ffvb676-2-e -force $project_name $project_dir

```
This create a `project_name.xpr` project file in the `project_dir` directory, wheere both of these arguments are 
passed from the makeile. 
```
$(VIVADO_PRJ_PATH):
	mkdir -p $(VIVADO_PRJ_DIR)
	$(VIVADO_CMD) setup.tcl -tclargs $(VIVADO_PRJ_DIR) $(VIVADO_PRJ_NAME)

setup: $(VIVADO_PRJ_PATH)
```

### Running implementation 


// trying it up with a makefile

## Generating the SVF file 

// some explaination of what the svf file is 

// some explaination as to why we needed to know the scan chain 

// why we need the bitstream 

// what the output looks like 

// tcl script presentation 

// why this is now vivado independant 



