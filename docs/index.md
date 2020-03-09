# Focal Documentation Site

Focal is a modern Fortran module library which wraps calls to the OpenCL runtime API (using [clfortran](https://github.com/cass-support/clfortran)) with a higher abstraction level appropriate to the Fortran language.

In particular, Focal removes all references to c pointers and provides compact but extensible subroutine wrappers to the OpenCL runtime API.
Moreover, Focal introduces typed buffer objects in host code which abstracts byte allocation away while providing built-in type safety.
Focal also provides a customisable error handler for OpenCL API errors as well as a [debug version](../errors#2-runtime-debug-checks) containing useful runtime checks for
ensuring OpenCL program validity.

__Project status:__ *Beta*

__Github:__ [github.com/lkedward/focal](https://github.com/LKedward/focal)

__License:__ [MIT](https://github.com/LKedward/focal/blob/master/LICENSE)

## Getting started

* [Building the Focal Library](build)
* [Using and linking Focal](linking)
* [Quickstart programming guide](quickstart)
* [Example programs](https://github.com/LKedward/focal/tree/master/examples)
* [Lattice Boltzmann demo](https://github.com/LKedward/lbm2d_opencl)

## A quick example
The following fortran program calculates the sum of two large arrays using an OpenCL kernel.

```fortran
program sum
!! Focal example program: calculate the sum of two arrays on an openCL device

use Focal
implicit none

integer, parameter :: Nelem = 1E6           ! No. of array elements
real, parameter :: sumVal = 10.0            ! Target value for array sum

integer :: i                                ! Counter variable
character(:), allocatable :: kernelSrc      ! Kernel source string
type(fclDevice), allocatable :: devices(:)      ! List of focal devices
type(fclProgram) :: prog                    ! Focal program object
type(fclKernel) :: sumKernel                ! Focal kernel object
real(c_float) :: array1(Nelem)              ! Host array 1
real(c_float) :: array2(Nelem)              ! Host array 2
type(fclDeviceFloat) :: array1_d            ! Device array 1
type(fclDeviceFloat) :: array2_d            ! Device array 2

! Create context with nvidia platform
call fclSetDefaultContext(fclCreateContext(vendor='nvidia'))

! Select device with most cores and create command queue
devices = fclFindDevices(sortBy='cores')
call fclSetDefaultCommandQ(fclCreateCommandQ(devices(1),enableProfiling=.true.))

! Load kernel from file and compile
call fclSourceFromFile('examples/sum.cl',kernelSrc)
prog = fclCompileProgram(kernelSrc)
sumKernel = fclGetProgramKernel(prog,'sum')

! Initialise device arrays
array1_d = fclBufferFloat(Nelem,read=.true.,write=.false.)
array2_d = fclBufferFloat(Nelem,read=.true.,write=.true.)

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
```openCL
__kernel void sum(const int nElem, const __global float * v1, __global float * v2){
  int i = get_global_id(0);
  if(i < nElem) v2[i] += v1[i];
}
```
