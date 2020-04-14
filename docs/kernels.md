# Working with kernels

See [setup](../setup) and [memory](../memory) first for how to setup contexts, command queues and manage device memory.

__See also in [advanced topics](../advanced):__

* [Local kernel arguments](../advanced#3-local-kernel-arguments)

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

In the call to `ld` the output binary file can have any arbitrary name, however the input file, containing the kernels source text, __MUST be called `fclKernels.cl` with no path prefix__.

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

See the examples [makefile](https://github.com/LKedward/focal/blob/master/examples/makefile) in the repository for how this can be done during the build process.

__API ref:__
[fclGetKernelResource](https://lkedward.github.io/focal-api/interface/fclgetkernelresource.html)

## 2. Compiling kernels

Once kernel source code has been loaded into a character string (`character(len=n)`), the code can be compiled using `fclCompileProgram`.
An OpenCL 'program' simply refers to a collection of kernels. So we can collect all our kernels into a single character string and compile them together.

__Interfaces__

```fortran
prog = fclCompileProgram([ctx],source,[options])
```
* `prog` (`type(fclProgram)`): compiled program object

* `ctx` (`type(fclContext)`,`optional`): context on which to compile program. Default context is used if omitted.

* `source` (`character(*)`): the OpenCL program source code to compile

* `options` (`character(*)`,`optional`): specifies [OpenCL compilation options](https://www.khronos.org/registry/OpenCL/sdk/1.2/docs/man/xhtml/clBuildProgram.html#notes).


Once compiled, a specific Focal kernel object can be extracted from the compiled program object using `fclGetProgramKernel`.
This returns an `fclKernel` object which we can use to launch a specific kernel.

__Interfaces__

```fortran
kern = fclGetProgramKernel(prog,kernelName,[global_work_size],[local_work_size], &
                                             [work_dim],[global_work_offset])
```

* `kern` (`type(fclKernel)`): kernel object used to launch kernels
 
* `prog` (`type(fclProgram)`): program object containing compiled kernels

* `kernelName` (`character(*)`): case-sensitive kernel function name

* `global_work_size` (`integer(dim)`,`optional`) array (up to length 3) specifying global work dimensions. Default unset.
If specified, also automatically sets `work_dim` to the length of `global_work_size`.

* `local_work_size` (`integer(dim)`,`optional`) array (up to length 3) specifying local work group dimensions.
Default [0,0,0] meaning OpenCL sets the local work dimensions.

* `work_dim` (`integer`,`optional`) specifies number of work group dimensions.
Default 1 or length of `global_work_size` if specified.

* `global_work_offset` (`integer(dim)`,`optional`) array specifying global work offsets in each dimension, default [0,0,0]

The kernel parameters `global_work_size`, `local_work_size`, `work_dim`, and `global_work_offset` can also be set using the
syntax:

```fortran
kern%global_work_size = [10, 10, 10]
kern%local_work_size = [10, 1, 1]
kern%work_dim = 3
kern%global_work_offset = [0, 0, 0]
```

!!! warning
    When kernels are launched, the `global_work_size` is automatically modified to ensure it is a multiple of `local_work_size` in
    all dimensions by increasing it where necessary - make sure to guard against out-of-bounds thread indexes within your kernels.

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

or equivalently:

```fortran
call fclLaunchKernel(myKernel,cmdq,arg1,arg2)
```

If the default command queue has been set, then `cmdq` can be omitted here.
In these examples two kernel arguments are supplied `arg1` and `arg2`.
Kernel arguments can be scalar variables, device buffer objects or local memory specifiers.
Up to 20 kernel arguments can be specified using this syntax.

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
    Use `fclSetKernelArg` and `fclSetKernelArgs` to change a single/multiple kernel argument(s) between kernel launches.


__API ref:__
[fclKernel](https://lkedward.github.io/focal-api/type/fclkernel.html),
[fclLaunchKernel](https://lkedward.github.io/focal-api/interface/fcllaunchkernel.html),
[fclLaunchKernelAfter](https://lkedward.github.io/focal-api/interface/fcllaunchkernelafter.html)
[fclSetKernelArg](https://lkedward.github.io/focal-api/interface/fclsetkernelarg.html),
[fclSetKernelArgs](https://lkedward.github.io/focal-api/interface/fclsetkernelargs.html)


