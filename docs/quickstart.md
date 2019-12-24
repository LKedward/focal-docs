# Quickstart Guide

See [build](../build) and [linking](../linking) first on how to build the library and subsequently link against it.

## 1. Create an OpenCL context

We can create a context using `fclCreateContext` by specifying a platform vendor:

```fortran
type(fclContext) :: ctx
...
ctx = fclCreateContext(vendor='nvidia')
```

If we're only using one platform/context, then we can shorten this by setting the default context (`fclDefaultCtx`) so that we don't need to pass a context variable to subsequent focal calls:


```fortran
call fclSetDefaultContext( fclCreateContext(vendor='nvidia') )
```

## 2. Select a device to work with

We can query devices based on their properties and return a list of them using `fclFindDevices'

```fortran
type(fclDevice), allocatable :: devices(:)
...
devices = fclFindDevices(ctx,type='gpu',nameLike='tesla',sortBy='cores')
```
`ctx` can be omitted here if the default context has been set.

This will look for devices in the current context and filter based on type ('cpu' or 'gpu') and device name.
An array of type(fclDevice) is returned where the devices have been sorted (descending) by some metric ('cores','memory','clock').
We can select the first device in this array and create a command queue on it:

```fortran
type(fclCommandQ) :: cmdq
...
cmdq = fclCreateCommandQ(devices(1))
```

Just like with the context, we can shorten this if we're only interested in using one command queue by setting the default command queue (`fclDefaultCmdQ`):

```fortran
call fclSetDefaultCommandQ( fclCreateCommandQ(devices(1) )
```
With this syntax, we can omit the command queue variable (`cmdq`) in subsequent focal calls.

## 3. Load and compile a kernel

A utility function `fclSourceFromFile` is provided to load text from file into an allocatable Fortran `character(:)` string:

```fortran
character(:), allocatable :: programSource
...
call fclSourceFromFile('kernels.cl',programSource)
```

Here `programSource` is a string containing a collection of OpenCL kernels. This program is then compiled, before extracting a Focal kernel object for the kernel we are interested in:

```fortran
type(fclProgram) :: prorg
type(fclKernel) :: myKernel
...
prog = fclCompileProgram(programSource)
myKernel = fclGetProgramKernel(prog,'kernelName')
```

## 4. Initialise device memory buffers

OpenCL kernels operate on memory residing on the device. We must therefore declare and initialise data buffers on the device.
In the following example we initialise a 32bit integer array with 1000 values on the command queue `cmdq`:

```fortran
type(fclDeviceInt32) :: deviceArray
...
deviceArray = fclBufferInt32(cmdq,dim=1000,read=.true.,write=.false.)
```

Again, `cmdq` can be omitted here if the default command queue has been set.

Other supported types are `fclDeviceFloat` and `fclDeviceDouble`.

The `read` and `write` arguments specify the read and write access of kernels running on the device.
In this example kernel can read this buffer, but not write to it.

Once the device buffers have been initialised, we can transfer data to them. In Focal this is done simply with an assignment statement:

```fortran
integer :: hostArray(1000)
...
deviceArray = hostArray
```

!!! note
    Focal transfer operations are only supported if the host array and device array are of compatible type and dimension .


In a similar way, data can be transferred from the device back to the host:

```fortran
hostArray = deviceArray
```

!!! note
    By default transfer operations involving a host array are __blocking__: host code does not continue until the transfer has completed.


## 5. Launch a kernel

We are now ready to launch a kernel. Using the kernel object `myKernel` extracted previously, we can set the global dimension which specifies the number of work items to launch:

```fortran
myKernel%global_work_size(1) = 1000
```
Here we specify a 1D work range with 1000 elements.

The kernel is can then be launched on command queue `cmdq` with the syntax:

```fortran
call myKernel%launch(cmdq,1000,deviceArray)
```

Again, `cmdq` can be omitted here if the default command queue has been set.

Here we have passed two arguments to the kernel: the scalar integer `1000` and the device array `deviceArray`.
