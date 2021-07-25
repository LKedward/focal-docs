# Managing memory

See [setup](../setup) first for how to define a context and create a device command queue.

__See also in [advanced topics](../advanced):__

* [Sub-buffers](../advanced#1-sub-buffers)
* [Pinned host memory](../advanced#2-pinned-host-memory)

## 1. Initialise device memory

Initialisation of device memory requires a dimension (number of elements) and logicaleans indicating the read/write access of kernels.
A command queue may optionally be specified, if omitted the the default command queue is used (`fclDefaultCommandQ`).


__Interfaces__
```fortran
call fclInitBuffer([cmdQ],untypedBuffer,nBytes,[profileName],[access])
call fclInitBuffer([cmdQ],typedBuffer,dim,[profileName],[access])
```

- `cmdQ` (*optional*) specifies the command queue to which this buffer is associated.
All buffer operations will occur on this command queue. `cmdQ` can be omitted if the default command queue has been set.

- `untypedBuffer` (`type(fclDeviceBuffer)`) is a general buffer object with no associated type information.

- `typedBuffer` can be one of `type(fclDeviceInt32)`, `type(fclDeviceFloat)`, `type(fclDeviceDouble)`.

- `nBytes` (`integer`): number of bytes to allocated for an untyped buffer object.

- `dim` (`integer`): number of elements for which to allocate the typed buffer.

- `profileName` (`character(*)`, `optional`): descriptive name for printing profiling information. See [profiling](../profiling).

- `access` (one of `'r'`, `'w'`, `'rw'`, `optional`): kernel read/write access to buffer, default `'rw'`.

!!! note
    All buffer operations are submitted to the command queue specified at buffer initialisation. To change the command
    queue associated with a buffer use the syntax: `buffer%cmdq => newCmdQ`.

__Example:__
read-only integer buffer with 1000 elements

```fortran
type(fclDeviceInt32) :: array_d
call fclInitBuffer(array_d,dim=1000,access='r')
```
No command queue is specified, so the default command queue is used (`fclDefaultCmdQ`) which is assumed to have been initialised.
'Read only' in this context means that OpenCL kernels cannot write to this buffer.

__Example:__
read-write double precision buffer with command queue

```fortran
type(fclDeviceDouble) :: array_d
type(fclCmdQ) :: cmdq
...
call fclInitBuffer(cmdq,array_d,dim=1000,profileName='array_d')
```

__API ref:__
[fclInitBuffer](https://lkedward.github.io/focal/interface/fclinitbuffer.html),
[fclDeviceBuffer](https://lkedward.github.io/focal/type/fcldevicebuffer.html),
[fclDeviceInt32](https://lkedward.github.io/focal/type/fcldeviceint32.html),
[fclDeviceFloat](https://lkedward.github.io/focal/type/fcldevicefloat.html),
[fclDeviceDouble](https://lkedward.github.io/focal/type/fcldevicedouble.html)



## 2. Fill device buffer with scalar

The assignment `=` operator can be used to fill an initialised device buffer with a scalar value.


__Example__
```fortran
type(fclDeviceFloat) :: deviceFloat
type(fclDevicDouble) :: deviceDouble
...
! Initialise device arrays
call fclInitBuffer(deviceFloat,Nelem)
call fclInitBuffer(deviceDouble,Nelem)

! Fill arrays with zeros
deviceFloat = 0.0
deviceDouble = 0.0d0
```

!!! note
    Since filling an array with a scalar value does not involve a host array pointer, it is __non-blocking__.
    The scalar assignment is enqueued onto the command queue and host code continues. Call `fclWait(cmdq%lastWriteEvent)` to wait.

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal/interface/assignment%28%3D%29.html)

## 3. Data transfer between device and host

Data transfer between a device buffer and a host array can be achieved using the assignment `=` operation.
When transferring data, the host and device arrays must be of compatible types.



__Example__:
transfer from host to device

```fortran
real(real32) :: hostArray(Nelem)
type(fclDeviceFloat) :: deviceArray
...
! Initialise device array
call fclInitBuffer(deviceArray,Nelem)

! Copy to device from host
deviceArray = hostArray
```

!!! note
    Since transferring arrays involves a host array pointer, it is __blocking__ by default.
    Host code does not continue until transfer completes. Set [command queue](../setup#2-creating-a-command-queue) `blockingWrite` and `blockingRead` to `.false.` for asynchronous transfers.


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
[Assignment(=)](https://lkedward.github.io/focal/interface/assignment%28%3D%29.html),
[fclCommandQ](https://lkedward.github.io/focal/type/fclcommandq.html),
[fclWait](https://lkedward.github.io/focal/interface/fclwait.html)


## 4. Transfer device array to device array

Device arrays and device array pointers can also be copied using the assignment `=` operator.

If both the source object (right value) and target object (left value) are valid initialised device array objects,
then the assignment operation will enqueue a __non-blocking__ device-to-device transfer.
This event can be waited upon by the host using `fclWait(cmdq%lastCopyEvent)`.

If the target object (left value) is not initialised then the assignment operation will copy the pointer from the initialised source object (right value)
such that both objects refer to the same device array.

If the source object (right value) is not initialised then the assignment operation is invalid and will result in a runtime error.

__Example__

```fortran
type(fclDeviceInt32) :: deviceArray1
type(fclDeviceInt32) :: deviceArray2
...
! Initialise device array 1
call fclInitBuffer(deviceArray1,Nelem)

deviceArray2 = deviceArray1
! deviceArray2 and deviceArray1 now reference the same device buffer
```

__Example__

```fortran
type(fclDeviceInt32) :: deviceArray1
type(fclDeviceInt32) :: deviceArray2
...
! Initialise both device arrays
call fclInitBuffer(deviceArray1,Nelem)
call fclInitBuffer(deviceArray2,Nelem)
...
deviceArray2 = deviceArray1
call fclWait(fclLastCopyEvent)
! Contents of deviceArray1 copied to deviceArray2
```

__API ref:__
[Assignment(=)](https://lkedward.github.io/focal/interface/assignment%28%3D%29.html),
[fclCommandQ](https://lkedward.github.io/focal/type/fclcommandq.html),
[fclWait](https://lkedward.github.io/focal/interface/fclwait.html)


## 5. Swap buffer pointers

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

## 6. Free device memory

Device memory is released using `fclFreeBuffer`.

__Example__

```fortran
call fclFreeBuffer(deviceArray)
```

__API ref:__
[fclFreeBuffer](https://lkedward.github.io/focal/interface/fclfreebuffer.html)


