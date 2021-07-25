# Advanced topics

## 1. Sub-buffers

Sub-buffers are separate buffer objects that reference a single contiguous slice of an existing device buffer.
Once initialised they behave like any other device buffer object.

__Interfaces:__
```fortran
call fclInitSubBuffer([cmdQ],untypedSubBuffer,sourceBuffer,offset,size, &
                        [profileName],[access])
call fclInitSubBuffer([cmdQ],typedSubBuffer,sourceBuffer,start,length, &
                        [profileName],[access])
```

- `cmdQ` (*optional*) specifies the command queue to which this buffer is associated.
All buffer operations will occur on this command queue. `cmdQ` can be omitted if the default command queue has been set.

- `untypedSubBuffer` (`type(fclDeviceBuffer)`): a general buffer object with no associated type information.

- `typedSubBuffer`: can be one of `type(fclDeviceInt32)`, `type(fclDeviceFloat)`, `type(fclDeviceDouble)`.

- `offset` (`integer`): the zero-based byte offset within the source buffer at which the sub-buffer starts.

- `size` (`integer`): the size in bytes of the sub-buffer.

- `start` (`integer`): the zero-based element offset within the source buffer at which the sub-buffer starts.

- `length` (`integer`): the number of elements in the sub-buffer.

- `profileName` (`character(*)`, `optional`): descriptive name for printing profiling information. See [profiling](../profiling).

- `access` (one of `'r'`, `'w'`, `'rw'`, `optional`): kernel read/write access to buffer, default `'rw'`.


__Example:__

```fortran
type(fclDeviceFloat) :: parentFloat, childFloat
...
call fclInitBuffer(parentFloat,10)
call fclInitSubBuffer(childFloat,parentFloat,5,5)
```

In this example, the `childFloat` sub-buffer references the second half of the `parentFloat` buffer which contains 10 elements in total.

__API ref:__
[fclInitSubBuffer](https://lkedward.github.io/focal/interface/fclinitsubbuffer.html)


## 2. Pinned host memory

*Pinned* memory refers to non-pageable memory allocated on the host; this memory can be accessed more directly by devices
since it is guaranteed not to have been '*paged-out*' by the host operating system.
Pinned memory therefore allows for both faster host-device transfers as well as being __required for asynchronous transfers__.

Pinned memory is implemented in OpenCL using the [clEnqueueMapBuffer](https://www.khronos.org/registry/OpenCL/sdk/1.2/docs/man/xhtml/clEnqueueMapBuffer.html) command
which maps a region of allocated '*host accessible*' memory to the host address space and returns a pointer to the host address.

### 2.1 Allocate pinned memory

To use pinned host memory in Focal, simply replace a standard dynamic allocation (*e.g.* `allocate(hostArray(N))`) with the `fclAllocHost` command:

__Interfaces:__

```fortran
call fclAllocHost(cmdQ,ptr,dim)
call fclAllocHost(ptr,dim)
```

`cmdQ` is the command queue onto which to enqueue the OpenCL map command.
As usual, when `cmdQ` is not specified the default command queue is assumed.

`ptr` is one of:

* `real(sp), pointer, dimension(:)`
* `real(dp), pointer, dimension(:)`
* `integer, pointer, dimension(:)`

where `sp` and `dp` refer to single precision and double precision kinds respectively.
Note that once allocated, `ptr` can be treated as any other `allocatable` Fortran array except that it cannot be deallocated using `deallocate`.

`dim` is the number of elements for which to allocate space for.

!!! note
    `fclAllocHost` is a blocking command: execution pauses on the host until the OpenCL map command has completed.

__Example:__
Allocate a one-dimensional float array of ten elements using pinned memory on the default command queue.

```fortran
real, pointer :: pinnedArray(:)
...
call fclAllocHost(pinnedArray,10)
```

__API ref:__
[fclAllocHost](https://lkedward.github.io/focal/interface/fclallochost.html)

### 2.2 Deallocate pinned memory

Pinned memory is implemented using pointers and is hence not automatically freed when it goes out of scope.
Therefore it __must be explicitly deallocated__ using `fclFreeHost` when no longer required.

__Interface:__

```fortran
call fclFreeHost(cmdq,ptr)
call fclFreeHost(ptr)
```

`cmdQ` is the command queue onto which to enqueue the OpenCL unmap command.
As usual, when `cmdQ` is not specified the default command queue is assumed.

where `ptr` is any pointer allocated using `fclAllocHost`.

!!! note
    `fclFreeHost` is a blocking command: execution pauses on the host until the OpenCL unmap command has completed.


__API ref:__
[fclFreeHost](https://lkedward.github.io/focal/interface/fclfreehost.html)


## 3. Local kernel arguments

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
[fclLocalInt32](https://lkedward.github.io/focal/interface/fcllocalint32.html),
[fclLocalFloat](https://lkedward.github.io/focal/interface/fcllocalfloat.html),
[fclLocalDouble](https://lkedward.github.io/focal/interface/fcllocaldouble.html),
[fclLocalArgument](https://lkedward.github.io/focal/type/fcllocalargument.html),


