+++ 
draft = false
date = 2018-12-28T18:46:01Z
title = "Writing makefiles for micro kernels"
slug = "kernel_makefile"
thumbnail = "images/profile.jpg"
description = ""
+++

This makefile contains the base for building tracer.
```bash
$ make clean
$ make upload
```

## Project overview 

Tracer's component are the following :

- `kernel` : the standard operationnal base, in charge of managing shared ressources.
	These resources can be hardware (processor, coprocessors, peripheral, memory, etc...) or software
	(files, shared mem, interfaces, etc...).
- `modules` : the variable operationnal base, executing at kernel privilege level, and taking advantage of
	the kernel's capabilities. Modules can be hardware dependant code, (peripheral drivers), code that must
	use abstracted hardware capabilities (servo controller).
- `apps` : processes, executing at low privilege, without access to kernel capabilities.
- `stdlib` : a set of standard functions, that can be used by all components.
	
## Building 

Each component is built using :

- A build folder, unique for each component, where its `.elf` objects must be generated.
- A set of dependency rules, that make units can populate. Those rules will compile code and store resulting
	objects in the component's build folder.
		
Components are built by successively including different make units. Each one  :

- defines make variables, to configurate other make units.
- implements compilation rules, each one being destinated to one component.
	Resulting objects should be generated in its build dir.
- add each rule to the appropriate component's list.
	
Five make units are included. In the following order :

- `hard` : contains standard components hardware dependent code. Its aim is to provide the most complete standard
	if for the whole projects. Non standard components (applications and projects mainly) are free to
	include hardware specific code.
	It configures the kernel, adds kernel rules to generate the kernel's hardware dependent parts, adds
	device driver modules, and generate hardware code for the standard lib (ex for math).
- `kernel` : builds the kernel's hardware independent part. Only has effect on the kernel component.
- `dmz` : contains utilities that are used by the kernel and by the stdlib. All source files in this make unit
	are compiled twice, once for the kernel, and once for the stdlib.
- `std` : contains the standard lib's hardware independent part. Only has an effect on stdlib component.
- `project` : generates application code, by creating and adding rules for the app and module component.

example : kernel Makefile
```makefile
#------------------------------------------- Components configuration flags --------------------------------------------

#Add the exception stacks size requirements;
KFLAGS += -DKEX_STACK_SIZE=$(KEX_STACK_SIZE)

#Add first process hardware requirements;
KFLAGS += -DKFP_RAM_SIZE=$(KFP_RAM_SIZE)
KFLAGS += -DKFP_STACK_SIZE=$(KFP_STACK_SIZE)
KFLAGS += -DKFP_ACTIVITY_TIME=$(KFP_ACTIVITY_TIME)

KFLAGS += -DKDM_SIZE=$(KDM_SIZE)

KFLAGS += -DNB_COPROCESSORS=$(KERNEL_NB_COPROCESSORS)


#-------------------------------------------------- Compilation rules --------------------------------------------------

KS := src/kernel
KR_BL := build/krnl

kernel :

	@$(KCC) -o $(KR_BL)/fault.o 		-c $(KS)/async/fault.c
	@$(KCC) -o $(KR_BL)/except.o 		-c $(KS)/async/except.c


	@$(KCC) -o $(KR_BL)/clock.o 		-c $(KS)/clock/clock.c
	@$(KCC) -o $(KR_BL)/sysclock.o 	-c $(KS)/clock/sysclock.c


	@$(KCC) -o $(KR_BL)/probe.o 		-c $(KS)/debug/probe.c
	@$(KCC) -o $(KR_BL)/debug.o 		-c $(KS)/debug/debug.c
	@$(KCC) -o $(KR_BL)/printk.o 			-c $(KS)/debug/printk.c


	@$(KCC) -o $(KR_BL)/inode.o 		-c $(KS)/fs/inode.c


	@$(KCC) -o $(KR_BL)/heap.o 		-c $(KS)/mem/heap.c
	@$(KCC) -o $(KR_BL)/kdmem.o 		-c $(KS)/mem/kdmem.c
	@$(KCC) -o $(KR_BL)/ram.o 			-c $(KS)/mem/ram.c


	@$(KCC) -o $(KR_BL)/mod.o 			-c $(KS)/mod/mod.c


	@$(KCC) -o $(KR_BL)/apps.o 			-c $(KS)/run/apps.c
	@$(KCC) -o $(KR_BL)/pmem.o 			-c $(KS)/run/pmem.c
	@$(KCC) -o $(KR_BL)/coproc.o 		-c $(KS)/run/coproc.c
	@$(KCC) -o $(KR_BL)/proc.o 		-c $(KS)/run/proc.c
	@$(KCC) -o $(KR_BL)/sched.o 		-c $(KS)/run/sched.c
	@$(KCC) -o $(KR_BL)/ksyscall.o 	-c $(KS)/ksyscall.c


	@$(KCC) -o $(KR_BL)/nlist.o 		-c $(KS)/struct/nlist.c
	@$(KCC) -o $(KR_BL)/shared_fifo.o 	-c $(KS)/struct/shared_fifo.c


	@$(KCC) -o $(KR_BL)/kinit.o 		-c $(KS)/kinit.c
	@$(KCC) -o $(KR_BL)/panic.o 		-c $(KS)/panic.c



#-------------------------------------------------- Components Config --------------------------------------------------

KRNL_RULES += kernel

```
### Linking barriers

As the executable is entirely static, all parts (kernel, modules, stdlib and apps), are linked together to form the
final executable. 

This raise a **security problem**. Indeed, modules and kernel run on a high privilleged level, and
stdlib and applications run on a lower privilege level. Their compilation and linking should be separated.

Moreover, modules and (resp) apps use public symbols of kernel and (resp) stdlib, but the opposite should never
happen. By no means the kernel or the stdlib should use a function of a module or an application.
There need to be a mechanism to restrict the linking process to only compatible parts of the code, but standard
toolchains do not provide such mechanism.

To achieve this, we will mangle (prefix) defined symbols of each object that would form the executable, and propagate
changes only to object that are authorised to be linked with it.

```makefile
#ELF mangle : translation_file,object_file
elf_mangle = $(OBJCOPY) --redefine-syms=$(1) $(2)
```

For example, we will mangle all public defined symbols of the kernel, and propagate this mangling to modules only.
That way, if the stdlib, or an application used a kernel symbol, during the final linking, it will query a symbol
that will have been mangled in the kernel object file but not in its object file. An undefined reference will exist.
This mechanism is only a safeguard, and doesn't fully prevent forbidden linking, as pre-using a mangled name
is sufficient to overpass it.

With such mechanisms, kernel and stdlib are stand_alone, and modules (resp applications) can only use their
symbols, and those of the kernel (resp stdlib).

> The only exception to this rule concerns two functions of the standard library, run_exec and run_exit, that are used by the kernel at a process's entry and exit points.

```makefile
#The standard lib is different. We must archive all objetcs, generate the translation table, and mangle std symbols;
stdl_pack : $(STDL_RULES)

#Archive the stdandard library;
	@$(AR) cr build/stdlib.a $(STDL_O)

#Generte the stdlib translation table;
	@$(call elf_translate,$(STDL_MGL_PREFIX),build/stdlib.a,build/stdl_trsl)

#Mangle stdlib's symbols in the stdlib archive;
@$(call elf_mangle,build/stdl_trsl,build/stdlib.a)
```

In opposition to kernel and stdlib that are unique, modules and applications are many, and their amound is not
defined. To register properly a module, or an application, create a rule that should build your objects,
and a variable that should contain your object files names.

 **Both rule and variable must have the same name**

The name itself is not important, as soon as it doesn't conflict with another rule / variable.

Finally, to include your module in the make process, add the rule / variable name in the appropriate rules
list. This list can be MODS_RULES or APPS_RULES.
