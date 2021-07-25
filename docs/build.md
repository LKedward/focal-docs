# Building the Focal Library


## Prerequisites

- [GNU make](https://www.gnu.org/software/make/) utility
- Fortran compiler supporting the 2008 standard (tested regularly with `gfortran` 7.4.0 & 9.1.0 and `ifort` 19.1.0 )
- An OpenCL development library (One of:
[Intel OpenCL SDK](https://software.intel.com/en-us/opencl-sdk),
[NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-downloads),
[AMD Radeon Software](https://www.amd.com/en/support) )

## Build
Navigate to the repository root and run make:

```shell
$> make -j
```

Parallel build is fully supported by the makefile, so use of the `-j` flag is recommended.

To build the example programs:

```shell
$> make examples
```

which will place example binaries in `path/to/focal/bin/`.

To build and run the tests:

```shell
$> make -j test
```

## Build within your own project

A non-recursive sub-makefile is [included](https://github.com/LKedward/focal/blob/master/make.include) in the Focal repository root directory
for inclusion within project makefiles.
To use in your own makefile:

1. Define the make variable `FOCAL_DIR` as the path to the Focal root directory
2. Include the sub-makefile into your project makefile (after the `all` recipe)
3. Create a dependency of your objects on `$(FOCAL_LIB_OBJS)`
4. Add `-I$(FOCAL_DIR)/mod/` to your Fortran compile flags
5. Add `-L$(FOCAL_DIR)/lib/ -lFocal -lOpenCL` to your linker flags

See [linking](../linking) for how to link against the Focal library.
See [this](https://github.com/LKedward/lbm2d_opencl) lattice Boltzmann code as an example.

__Example:__

```makefile

FC ?= gfortran
FOCAL_DIR=./focal
PROGS= myProgram
OBJS= $(addsuffix .o, $(PROGS))

FFLAGS+= -I$(FOCAL_DIR)/mod/
LFLAGS+= -L$(FOCAL_DIR)/lib/ -lFocal -lOpenCL

all: $(PROGS)

include $(FOCAL_DIR)/make.include

# Objects depend on focal library
$(OBJS): $(FOCAL_LIB_OBJS)

# Recipe to link executable(s)
$(PROGS): $(OBJS)
    $(FC) $^ $(LFLAGS) -o $@

# Recipe to compile objects
%.o: %.f90
    $(FC) $(FFLAGS) -c $< -o $@


```
