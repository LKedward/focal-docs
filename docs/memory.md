# Managing memory

## Initialise device memory

Initialisation of device memory requires a dimension (number of elements) and booleans indicating the read/write access of kernels.
A command queue may optionally be specified, if omitted the the default command queue is used (`fclDefaultCommandQ`).


__Interfaces__
```fortran
<fclDeviceInt32> = fclBufferInt32(<fclCommandQ>,dimension,read,write)
<fclDeviceInt32> = fclBufferInt32(dimension,read,write)
<fclDeviceFloat> = fclBufferFloat(<fclCommandQ>,dimension,read,write)
<fclDeviceFloat> = fclBufferFloat(dimension,read,write)
<fclDeviceDouble> = fclBufferDouble(<fclCommandQ>,dimension,read,write)
<fclDeviceDouble> = fclBufferDouble(dimension,read,write)
```

__Example: read-only integer buffer with 1000 elements__

```fortran
type(fclDeviceInt32) :: array_d
array_d = fclBufferInt32(dim=1000,read=.true.,write=.false.)
```
No command queue is specified, so the default command queue is used (`fclDefaultCmdQ`) which is assumed to have been initialised.
'Read only' in this context means that OpenCL kernels cannot write to this buffer.

__Example: read-write double precision buffer with command queue__

```fortran
type(fclDeviceDouble) :: array_d
type(fclCmdQ) :: cmdq
...
array_d = fclBufferDouble(cmdq,dim=1000,read=.true.,write=.true.)
```

## Data transfer between device and host

Data transfer between a device buffer and a host array can be achieved with a simple assignment `=` operation.
When transferring data, the host and device arrays must be of compatible types.
Data transfers involving a host array are __blocking__ by default: host code does not continue until the transfer has completed.

__Interfaces__

```fortran
<fclDeviceInt32(n)>  = <integer>      ! host_scalar-to-device
<fclDeviceFloat(n)>  = <float>
<fclDeviceDouble(n)> = <double>

<fclDeviceInt32(n)>  = <integer(n)>   ! Host-to-device
<fclDeviceFloat(n)>  = <real32(n)>   
<fclDeviceDouble(n)> = <real64(n)>

<integer(n)> = <fclDeviceInt32(n)>    ! Device-to-host
<real32(n)> = <fclDeviceFloat(n)>
<real64(n)> = <fclDeviceDouble(n)>
```

__Example__

```fortran
real(real32) :: hostArray(Nelem)
type(fclDeviceFloat) :: deviceArray
...
! Initialise device array
deviceArray = fclBufferFloat(Nelem,read=.true.,write=.true.)

! Copy to device from host
deviceArray = hostArray
```

__Example: Non-blocking data transfer__

To perform non-blocking transfer, we set the Focal global parameter variables `fclBlockingWrite` and `fclBlockingRead` to `.false.` as required.
To keep track of the transfer operations we can use the Focal global event variables `fclLastWriteEvent` and `fclLastReadEvent`.

```fortran
type(c_ptr) :: e
...
fclBlockingWrite = .false.
deviceArray = hostArray
e = fclLastWriteEvent
...
call fclWait(e)
```