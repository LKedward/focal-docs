# Quickstart Guide

See [build](../build) and [linking](../linking) first on how to build the library and subsequently link against it.

## 1. Choose a device and initialise OpenCL context

We can quickly initialse an OpenCL context with a specific device using the `fclInit` function.
This functions allows us to filter and sort available devices based on their properties and choose one of them
to work with.

```fortran
type(fclDevice) :: device
...
device = fclInit(vendor='nvidia,amd',type='gpu',sortBy='cores')
```

In this example we have specified any `gpu` device having belonging to vendors `nvidia` OR `amd` and to choose the device
with the most compute units (`cores`).

Additional filter fields that can be used are: `extensions` to filter based on support OpenCL extensions;
and `nameLike` to filter based on the name of the device.

If the function is able to find a device matching the criteria, then it initialises an OpenCL context on the corresponding
platform and returns an `fclDevice` object on which we can create command queues.

See [User guide/Setup](../setup/) for more detail on querying available platforms and devices with Focal and advanced methods for setting up OpenCL contexts and using multiple devices.

## 2. Create a command queue
Once we have an `fclDevice` object for our chosen device, we can create a command queue on it which will be used to issue operations to device:

```fortran
type(fclDevice) :: device
type(fclCommandQ) :: cmdq
...
cmdq = fclCreateCommandQ(device)
```

The resulting command queue object `cmdq` can be passed to subsequent commands to specify which command queue to use.
However a common command queue called the __default command queue__ exists within Focal which when set allows you to omit
the command queue object in subsequent Focal commands.

To create a command queue and set as the default command queue:

```fortran
call fclSetDefaultCommandQ( fclCreateCommandQ(device) )
```

With this syntax, we can omit the command queue variable (`cmdq`) in subsequent focal calls.

See [User guide/Setup](../setup/) for more details on creating command queues.

## 3. Load and compile a kernel

A utility function `fclSourceFromFile` is provided to load text from file into an allocatable Fortran `character(:)` string:

```fortran
character(:), allocatable :: programSource
...
call fclSourceFromFile('kernels.cl',programSource)
```

Here `programSource` is a string containing a collection of OpenCL kernels.
OpenCL programs are compiled at runtime - this allows perfect portability since the kernels will always be compiled
for the local available hardware.

In Focal we use the `fclCompileProgram` command to compile the OpenCL program before using the `fclGetProgramKernel` 
to extract a kernel object for the particular kernel we are interested in:

```fortran
type(fclProgram) :: prorg
type(fclKernel) :: myKernel
...
prog = fclCompileProgram(programSource)
myKernel = fclGetProgramKernel(prog,'kernelName',global_work_size=[1000])
```

The optional third argument specifies the global work dimensions for the kernel; here we have specified
a one-dimensional global work set with 1000 elements.
Additional optional arguments here can include `local_work_size`, `work_dim` and `global_work_offset`.

Additionally, these kernel attributes can be set using the syntax:

```fortran
myKernel%global_work_size = [100,100]
myKernel%work_dim = 2
myKernel%local_work_size = [10,10]
myKernel%global_work_offset = [50,50]
```

See [User guide/Kernels](../kernels/) for more details on compiling and extracting kernels.


## 4. Initialise device memory buffers

OpenCL kernels operate on memory residing on the device. We must therefore declare and initialise data buffers on the device.
In the following example we initialise a 32bit integer array with 1000 values on the command queue `cmdq`:

```fortran
type(fclDeviceInt32) :: deviceArray
...
call fclInitBuffer(cmdq,deviceArray,dim=1000)
```

!!! note
    All future buffer operations (transfers to/from device and copying on device) will be issued to the command queue `cmdq` specified here.

If the default command queue has been set and you wish to use it for this buffer, then `cmdq` can be omitted in the call to `fclInitBuffer`:

```fortran
type(fclDeviceInt32) :: deviceArray
...
call fclInitBuffer(deviceArray,dim=1000)
```

Other supported types are `fclDeviceFloat` and `fclDeviceDouble`.

Once the device buffers have been initialised, we can transfer data to them. In Focal this is done simply with an assignment statement:

```fortran
integer :: hostArray(1000)
...
deviceArray = hostArray
```

!!! note
    Focal transfer operations are only supported if the host array and device array are of compatible type and dimension.
    If the host array and device array are not of compatible type, then a compile-time error will occur.
    If the host array and device array are not of the same dimension, then a run-time error will occur.


In a similar way, data can be transferred from the device back to the host:

```fortran
hostArray = deviceArray
```

as well as copying between two buffers residing on the device:

```fortran
type(fclDeviceFloat) :: deviceArray1, deviceArray2
...
deviceArray1 = deviceArray2
```

!!! note
    By default transfer operations involving a host array are __blocking__: host code does not continue until the transfer has completed.


## 5. Launch a kernel

We are now ready to launch a kernel. Using the kernel object `myKernel` extracted previously, this can then be launched on command queue `cmdq` with the syntax:

```fortran
call myKernel%launch(cmdq,1000,deviceArray)
```

Again, `cmdq` can be omitted here if you wish to use the default command queue assuming it has been set.

Here we have passed two arguments to the kernel: the scalar integer `1000` and the device array `deviceArray`.
