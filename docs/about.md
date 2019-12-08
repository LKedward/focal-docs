# About

The goal of Focal is to provide a concise and accessible Fortran interface to the OpenCL API while retaining the full functionality thereof.
This is desirable in Fortran which as a language provides a higher level of abstraction than C; importantly this allows scientists and engineers to focus on their domain specific problem rather than details of low-level implementation.

The aims for Focal in particular are to:

- remove the need to use c pointers in Fortran for the OpenCL API;
- provide some level of type-safety through the use of typed buffer objects;
- decrease the verbosity of the API calls while still providing the same functionality;
- abstract away low-level details, such as buffer size in bytes, not appropriate to Fortran;
- make it easier to write and debug OpenCL programs.

Focal officially supports OpenCL v1.2 features.

## Project status

The Focal project is __Beta__ development:

* All major features are implemented and documented

ToDo:

* Sub-buffer creation
* Allow multiple vendors to be specified in fclCreateContext
* Test suite
* Example codes

## See also

* [Khronos group](https://www.khronos.org/opencl/)
* [OpenCL specification](https://www.khronos.org/registry/OpenCL/specs/opencl-1.2.pdf)
* [Hands on OpenCL](https://handsonopencl.github.io/) (Complete course with code examples)
* [clfortran project](https://github.com/cass-support/clfortran) (Fortran interface header library)
* [NVIDIA OpenCL Best Practices](https://www.nvidia.com/content/cudazone/CUDABrowser/downloads/papers/NVIDIA_OpenCL_BestPracticesGuide.pdf)
* [NVIDIA OpenCL programming](https://www.nvidia.com/content/cudazone/CUDABrowser/downloads/papers/NVIDIA_OpenCL_BestPracticesGuide.pdf)
* [Intetl OpenCl optimization guide for intel graphics](https://software.intel.com/en-us/iocl-opg)
* [AMD OpenCL guide](https://rocm-documentation.readthedocs.io/en/latest/Programming_Guides/Opencl-programming-guide.html)
