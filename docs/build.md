# Building the Focal Library
To use the Focal library, it must first be compiled for the target architecture.

The Focal library is built and tested regularly with gfortran on RHEL, Ubuntu and Windows (using MinGW under [MSYS2](https://www.math.ucla.edu/~wotaoyin/windows_coding.html)).

## Prerequisies
- Fortran compiler supporting the 2008 standard (tested with gfortran and ifort)
- An OpenCL development library (One of:
[Intel OpenCL SDK](https://software.intel.com/en-us/opencl-sdk),
[NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-downloads),
[AMD OpenCL SDK](https://github.com/GPUOpen-LibrariesAndSDKs/OCL-SDK/releases) )

## Build
Navigate to the repository root and run make:

```shell
$> make -j
```

Parallel build is fully supported by the makefile, so use of the `-j` flag is recommended.

If the linker fails to link `-lOpenCL`, you will have to specify the location of the OpenCL development library manually, e.g.:

```shell
$> make -j OPENCL_DIR="path/to/OpenCL/"
```

To build the example programs:

```shell
$> make examples
```

which will place example binaries in `$(FOCAL_DIR)/bin/`.

### Build within your own project

A non-recursive sub-makefile is [included](https://github.com/LKedward/focal/blob/master/make.include) in the Focal repository root directory
for inclusion within project makefiles.
To use, simply include the sub-makefile into your project makefile (after the `all` recipe) and define the make variable `FOCAL_DIR` as the path to the Focal root directory;
finally create a dependency of your executable(s) on `$(FOCAL_LIB_OBJS)`.
This will ensure the Focal library is built prior to executable linking.
See [linking](../linking) for how to link against the Focal library.

Consider including Focal within your project using [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).
