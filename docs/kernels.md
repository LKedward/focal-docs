# Working with kernels

See [setup](../setup) and [memory](../memory) first for how to setup contexts, command queues and manage device memory.

## 1. Loading kernel sources

OpenCL kernels are almost always shipped as plain text sources which are compiled at runtime for the required architecture; 
this allows perfect portability of programs using OpenCL kernels.

Focal provides two utility routines for obtaining OpenCL kernel source code at runtime:

- `fclSourceFromFile` which loads source code from a file path
- `fclGetKernelResource` which loads source code that was linked at compile time as a binary resource.

Loading source using `fclSourceFromFile` from allows kernel sources to be updated without recompilation however it relies on robustly being able to locate kernel source files at runtime.

Linking source code as a binary resource allows kernel sources to be included into the program executable and therefore doesn't require source files to be distributed in addition to the OpenCL program.

### 1.1 From file

Use `fclSourceFromFile` to load source code from a file at runtime.

__Example__

```fortran
character(:), allocatable :: programSource
...
call fclSourceFromFile(filename='programKernels.cl',programSource)
```

In this example, kernel source code is read from the file `programKernels.cl` into the character string `programSource` which is allocated during the call.

__API ref:__
[fclSourceFromFile](https://lkedward.github.io/focal-api/interface/fclsourcefromfile.html)

### 1.2 From linked binary resource

__Step 1:__
Create object file from kernel code.

At the command line we use the linker `ld` to convert the kernel text file `fclKernels.cl` into a binary resource object:

```shell
$> ld -r -b binary fclKernels.cl -o fclKernels.o
```

In the call to `ld` the file containing the kernels text can have any arbitrary name, however the resulting binary object __MUST be called `fclKernels.o` with no path prefix__.

__Step 2:__
Compile and link the main program with the `fclKernels.o` resource object:

```shell
$> gfortran -c myCLProgram.f90 -o myCLProgram.o
$> gfortran myCLProgram.o fclKernels.o -lfocal -lOpenCL -o myCLProgram
```

__Step 3:__
Load the linked kernel source at runtime using `fclGetKernelResource`

```fortran
character(:), allocatable :: programSource
...
call fclGetKernelResource(programSource)
```

See examples in the repository for how this can be done with a makefile.

__API ref:__
[fclGetKernelResource](https://lkedward.github.io/focal-api/interface/fclgetkernelresource.html)

## 2. Compiling kernels

Once kernel source code has been loaded into a character string (`character(len=n)`), the code can be compiled using `fclCompileProgram`.
An OpenCL 'program' simply refers to a collection of kernels. So we can collect all our kernels into a single character string and compile them together.
Once compiled, a specific Focal kernel object can be extracted from the compiled program object using `fclGetProgramKernel`.
This returns an `fclKernel` object which we can use to launch a specific kernel.

__Interfaces__

```fortran
<fclProgram> = fclCompileProgram(ctx=<fclContext>,source=<character(len=n)>,options=<character(len=n)>)
```

```fortran
<fclKernel> = fclGetProgramKernel(<fclProgram>,kernelName=<character(len=n)>)
```

If the default context has been set then `ctx` can be omitted.

The `options` argument is optional and specifies [OpenCL compilation options](https://www.khronos.org/registry/OpenCL/sdk/1.2/docs/man/xhtml/clBuildProgram.html#notes).

__Example__


```fortran
type(fclProgram) :: program
type(fclKernel) :: myKernel
...
program = fclCompileProgram(myProgramSource)
myKernel = fclGetProgramKernel(program,'myKernel')
```

__API ref:__
[fclCompileProgram](https://lkedward.github.io/focal-api/interface/fclcompileprogram.html),
[fclGetProgramKernel](https://lkedward.github.io/focal-api/interface/fclgetprogramkernel.html),
[fclProgram](https://lkedward.github.io/focal-api/type/fclprogram.html),
[fclKernel](https://lkedward.github.io/focal-api/type/fclkernel.html)




## 3. Launching kernels

Once an `fclKernel` object exists, the kernel can be launched with the following syntax:

```fortran
myKernel%launch(cmdq,arg1,arg2)
```

If the default command queue has been set, then `cmdq` can be omitted here.
In this example two kernel arguments are supplied `arg1` and `arg2`.
Kernel arguments can be scalar variables, device buffer objects or local memory specifiers.

__Example:__
Launch a kernel with a scalar integer argument and a device buffer argument.

```fortran
integer :: nElem = 1000
type(fclDeviceInt32) :: deviceArray
...
deviceArray = fclBufferInt32(dim=nElem,read=.true.,write=.true.)
myKernel%launch(nElem,deviceArray)
```

!!! note
    OpenCL kernel arguments persist between calls, therefore subsequent kernel launch calls can omit arguments if unchanged.


__API ref:__
[fclKernel](https://lkedward.github.io/focal-api/type/fclkernel.html), 
[fclLaunchKernel](https://lkedward.github.io/focal-api/interface/fcllaunchkernel.html)

### 3.1 Kernel dimensions
We can launch 2 and 3 dimensional work arrays by specifying the number of dimensions of the range, and the dimension sizes:

```fortran
myKernel%dim = 2
myKernel%global_work_size(1:2) = [1000, 5000]
```

We can also specify the dimensions of the local work groups:

```fortran
myKernel%local_work_size=[20, 50]
```


### 3.2 Local memory arguments

If a kernel has a local memory argument, then the functions `fclLocalInt32`, `fclLocalFloat` and `fclLocalDouble` can be used to
supply an argument with a specific dimension.
Local memory arguments don't pass data to the kernel, they act like temporary variable length arrays where the size of the array can be specified at kernel launch time.

__Example__

The following OpenCL kernel has a local memory argument as the third argument:

```c
__kernel void myKernel(const int size, __global float * vec, __local float * temp){
  int ii = get_global_id(0);
  int jj = get_local_id(0);
  temp[jj] = vec[ii]
  ...
}
```

To launch this kernel with Focal, we use:

```fortran
myKernel%launch(nElem,deviceArray, fclLocalFloat(localSize) )
```

where `localSize` specifies the size of the local argument float array.

__API ref:__
[fclLocalInt32](https://lkedward.github.io/focal-api/interface/fcllocalint32.html),
[fclLocalFloat](https://lkedward.github.io/focal-api/interface/fcllocalfloat.html),
[fclLocalDouble](https://lkedward.github.io/focal-api/interface/fcllocaldouble.html),
[fclLocalArgument](https://lkedward.github.io/focal-api/type/fcllocalargument.html),

