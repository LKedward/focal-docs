# Managing memory

See [setup](../setup) first for how to define a context and create a device command queue.

## 1. Initialise device memory

Initialisation of device memory requires a dimension (number of elements) and logicaleans indicating the read/write access of kernels.
A command queue may optionally be specified, if omitted the the default command queue is used (`fclDefaultCommandQ`).


__Interfaces__
```fortran
<fclDeviceInt32> = fclBufferInt32([cmdQ],dim=<int>,read=<logical>,write=<logical>,[profileSize],[profileName])
<fclDeviceFloat> = fclBufferFloat([cmdQ],dim=<int>,read=<logical>,write=<logical>,[profileSize],[profileName])
<fclDeviceDouble> = fclBufferDouble([cmdQ],dim=<int>,read=<logical>,write=<logical>,[profileSize],[profileName])
```

- `cmdQ` (*optional*) specifies the command queue to which this buffer is associated.
All buffer operations will occur on this command queue. `cmdQ` can be omitted if the default command queue has been set.

- `dim` specifies the number of elements for which to allocate the buffer.

- `read` specifies whether kernels can read from this buffer

- `write` specifies whether kernels can write to this buffer

- `profileSize` (*optional*) specifies the number of data transfer events to 'record' for profiling.
Default 0 (profiling disabled). See [profiling](../profiling).

- `profileName` (*optional*) descriptive name when printing profiling information. See [profiling](../profiling).

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

__API ref:__
[fclBufferInt32](https://lkedward.github.io/focal-api/interface/fclbufferint32.html),
[fclBufferFloat](https://lkedward.github.io/focal-api/interface/fclbufferfloat.html),
[fclBufferDouble](https://lkedward.github.io/focal-api/interface/fclbufferdouble.html)



## 2. Fill device buffer with scalar

The assignment `=` operator can be used to fill an initialised device buffer with a scalar value.

__Interfaces__
```fortran
<fclDeviceInt32(n)>  = <integer>      ! host_scalar-to-device
<fclDeviceFloat(n)>  = <float>
<fclDeviceDouble(n)> = <double>
```

__Example__
```fortran
type(fclDeviceFloat) :: deviceFloat
type(fclDevicDouble) :: deviceDouble
...
! Initialise device arrays
deviceFloat = fclBufferFloat(Nelem,read=.true.,write=.true.)
deviceDouble = fclBufferDouble(Nelem,read=.true.,write=.true.)

! Fill arrays with zeros
deviceFloat = 0.0
deviceDouble = 0.0d0
```

!!! note
    Since filling an array with a scalar value does not involve a host array pointer, it is __non-blocking__.
    The scalar assignment is enqueued onto the device and host code continues. Call `fclWait(cmdq%lastWriteEvent)` to wait.

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal-api/interface/assignment%28%3D%29.html)

## 3. Data transfer between device and host

Data transfer between a device buffer and a host array can be achieved using the assignment `=` operation.
When transferring data, the host and device arrays must be of compatible types.

### 3.1 Transfer from host to device


__Interfaces__
```fortran
<fclDeviceInt32(n)>  = <integer(n)>   ! Host-to-device
<fclDeviceFloat(n)>  = <real32(n)>   
<fclDeviceDouble(n)> = <real64(n)>
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

!!! note
    Since transferring arrays involves a host array pointer, it is __blocking__ by default.
    Host code does not continue until transfer completes. Set `cmdq%blockingWrite = .false.` for asynchronous transfer.

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal-api/interface/assignment%28%3D%29.html)


### 3.2 Transfer from device to host

__Interfaces__
```fortran
<integer(n)> = <fclDeviceInt32(n)>    ! Device-to-host
<real32(n)> = <fclDeviceFloat(n)>
<real64(n)> = <fclDeviceDouble(n)>
```

!!! note
    Since transferring arrays involves a host array pointer, it is __blocking__ by default.
    Host code does not continue until transfer completes. Set `cmdq%blockingRead = .false.` for asynchronous transfer.

__Example: Non-blocking data transfer__

To perform non-blocking transfer, we set the command queue parameter variables `blockingWrite` and `blockingRead` to `.false.` as required.
To keep track of the transfer operations we can use the command queue variables `lastWriteEvent` and `lastReadEvent` or the global variables `fclLastWriteEvent` and `fclLastReadEvent`.

```fortran
type(fclEvent) :: e
...
cmdq%blockingWrite = .false.    ! Enable non-blocking host-to-device transfers
deviceArray = hostArray         ! Enqueue the transfer command
e = cmdq%lastWriteEvent         ! Get transfer event object
...
call fclWait(e)                 ! Wait for event when needed
```

If using the default command queue then replace `cmdq` with `fclDefaultCmdQ`.

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal-api/interface/assignment%28%3D%29.html),
[fclCommandQ](https://lkedward.github.io/focal-api/type/fclcommandq.html),
[fclWait](https://lkedward.github.io/focal-api/interface/fclwait.html)


## 4. Transfer device array to device array

Device arrays and device array pointers can also be copied using the assignment `=` operator.

__Interfaces__
```fortran
<fclDeviceInt32(n)> = <fclDeviceInt32(n)>    ! Device-to-device
<fclDeviceFloat(n)> = <fclDeviceFloat(n)>
<fclDeviceDouble(n)> = <fclDeviceDouble(n)>
```

If both the source object (right value) and target object (left value) are valid initialised device array objects,
then the assignment operation will enqueue a __non-blocking__ device-to-device transfer.
This event can be waited upon using `fclWait(cmdq%lastCopyEvent)`.

If the target object (left value) is not initialised then the assignment operation will copy the pointer from the initialised source object (right value)
such that both objects refer to the same device array.

If the source object (right value) is not initialised then the assignment operation is invalid and throws a runtime error.

__Example__

```fortran
type(fclDeviceInt32) :: deviceArray1
type(fclDeviceInt32) :: deviceArray2
...
! Initialise device array 1
deviceArray1 = fclBufferInt32(Nelem,read=.true.,write=.true.)

deviceArray2 = deviceArray1
! deviceArray2 and deviceArray1 now reference the same device buffer
```

__Example__

```fortran
type(fclDeviceInt32) :: deviceArray1
type(fclDeviceInt32) :: deviceArray2
...
! Initialise both device arrays
deviceArray1 = fclBufferInt32(Nelem,read=.true.,write=.true.)
deviceArray2 = fclBufferInt32(Nelem,read=.true.,write=.true.)
...
deviceArray2 = deviceArray1
call fclWait(fclLastCopyEvent)
! Contents of deviceArray1 copied to deviceArray2
```

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal-api/interface/assignment%28%3D%29.html),
[fclCommandQ](https://lkedward.github.io/focal-api/type/fclcommandq.html),
[fclWait](https://lkedward.github.io/focal-api/interface/fclwait.html)


## 5. Free device memory

Device memory is released using `fclFreeBuffer`.

__Example__

```fortran
call fclFreeBuffer(deviceArray)
```

__API ref:__
[fclFreeBuffer](https://lkedward.github.io/focal-api/interface/fclfreebuffer.html)


## 6. Swap buffer pointers

Device buffer pointers can be swapped on the host using `fclBufferSwap`.
This can be more efficient than performing a device-to-device copy.

__Example__

```fortran
integer :: i
type(fclKernel) :: myKernel
type(fclDeviceFloat) :: a1_d, a2_d
...
do i=1,10
  call myKernel%launch(a1_d)
  ...
  call fclBufferSwap(a1_d,a2_d)
end do

```
