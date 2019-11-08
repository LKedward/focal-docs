# Building the Focal Library
To use the Focal library, it must first be compiled for the target architecture.


## Prerequisies
- Fortran compiler supporting the 2008 standard (tested with gfortran and ifort)
- An openCL development library (One of:
[Intel OpenCL SDK](https://software.intel.com/en-us/opencl-sdk), 
[NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-downloads), 
[AMD OpenCL SDK](https://github.com/GPUOpen-LibrariesAndSDKs/OCL-SDK/releases) )

## Build
Navigate to the repository root and run make:

```sh
make -j
```

If the linker fails to link `-lOpenCL`, you will have to specify the location of the openCL development library manually, e.g.:

```sh
make -j OPENCL_DIR="path/to/OpenCL/"
```

To build the example programs:

```sh
make -j examples
```
