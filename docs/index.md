# Focal Documentation Site

Focal is a modern Fortran module library which wraps calls to the OpenCL runtime API (using [clfortran](https://github.com/cass-support/clfortran)) with a higher abstraction level appropriate to the Fortran language.

In particular, Focal removes all references to c pointers and provides compact but extensible subroutine wrappers to the OpenCL runtime API.
Moreover, Focal introduces typed buffer objects in host code which abstracts byte allocation away while providing built-in type safety.
Focal also provides a customisable error handler for OpenCL API errors as well as a [debug version](./errors#2-runtime-debug-checks) containing useful runtime checks for
ensuring OpenCL program validity.

__Project status:__ v1.0.1 Stable release

__Github:__ [github.com/lkedward/focal](https://github.com/LKedward/focal)

__License:__ [MIT](https://github.com/LKedward/focal/blob/master/LICENSE)

__Prerequisites:__

- [GNU make](https://www.gnu.org/software/make/) utility
- Fortran compiler supporting the 2008 standard (tested regularly with `gfortran` 7.4.0 & 9.1.0 and `ifort` 19.1.0 )
- An OpenCL development library (One of:
[Intel OpenCL SDK](https://software.intel.com/en-us/opencl-sdk),
[NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-downloads),
[AMD Radeon Software](https://www.amd.com/en/support) )


## Getting started

* [Building the Focal Library](build)
* [Using and linking Focal](linking)
* [Quickstart programming guide](quickstart)
* [Example programs](https://github.com/LKedward/focal/tree/master/examples)
* [Lattice Boltzmann demo](https://github.com/LKedward/lbm2d_opencl)


## Main features

* Removes use of c pointers to call OpenCL API
* Provides a level of type safety using typed buffer objects
* Decreases verbosity of OpenCL API calls while still providing the same functionality
* Abstracts away low level details, such as size in bytes
* Contains built-in customisable error handling for all OpenCL API calls
* Contains built-in 'debug' mode for checking program correctness
* Contains build-in routines for collecting and presented profiling information


## Quick example
The following Fortran program calculates the sum of two large arrays using an OpenCL kernel.

```fortran
program sum
!! Focal example program: calculate the sum of two arrays on an openCL device

use Focal
implicit none

integer, parameter :: Nelem = 1E6           ! No. of array elements
real, parameter :: sumVal = 10.0            ! Target value for array sum

integer :: i                                ! Counter variable
character(:), allocatable :: kernelSrc      ! Kernel source string
type(fclDevice) :: device                   ! Device object
type(fclProgram) :: prog                    ! Focal program object
type(fclKernel) :: sumKernel                ! Focal kernel object
real :: array1(Nelem)                       ! Host array 1
real :: array2(Nelem)                       ! Host array 2
type(fclDeviceFloat) :: array1_d            ! Device array 1
type(fclDeviceFloat) :: array2_d            ! Device array 2

! Select device with most cores and create command queue
device = fclInit(vendor='nvidia',sortBy='cores')
call fclSetDefaultCommandQ(fclCreateCommandQ(device,enableProfiling=.true.))

! Load kernel from file and compile
call fclSourceFromFile('examples/sum.cl',kernelSrc)
prog = fclCompileProgram(kernelSrc)
sumKernel = fclGetProgramKernel(prog,'sum')

! Initialise device arrays
call fclInitBuffer(array1_d,Nelem)
call fclInitBuffer(array2_d,Nelem)

! Initialise host array data
do i=1,Nelem
  array1(i) = i
end do
array2 = sumVal - array1

! Copy data to device
array1_d = array1
array2_d = array2

! Set global work size equal to array length and launch kernel
sumKernel%global_work_size(1) = Nelem
call sumKernel%launch(Nelem,array1_d,array2_d)

! Copy result back to host and print out to check
array2 = array2_d
write(*,*) array2(1), array2(size(array2,1))

end program sum
```

Where `sum.cl` contains the following openCL kernel:
```c
__kernel void sum(const int nElem, const __global float * v1, __global float * v2){
  int i = get_global_id(0);
  if(i < nElem) v2[i] += v1[i];
}
```
