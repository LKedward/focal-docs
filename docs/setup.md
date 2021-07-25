# Setup

All OpenCL actions occur via contexts, which are containers for related devices, and command queues which are attached to specific devices.
Each context is associated with a specific platform where a platform generally coincides with a particular vendor.


## 1a. Quick setup 

We can use the `fclInit` function to quickly select a device based on criteria from all devices available on the system.
If a device matching the specified criteria is found, then this device is returned as an object and the __default context__ is set
using the platform containing the matching device.

__Interface:__

```fortran
device = fclInit([vendor],[type],[nameLike],[extensions],[sortBy])
```

* `device` (`type(fclDevice)`): the chosen device returned by the function.

* `vendor` (`character(*)`,`optional`): if specified, look for this (sub)string in the device vendor to filter available devices.

* `type` (one of `'cpu'` or `'gpu'`,`optional`): if specified, filter available devices based on device type.

* `nameLike` (`character(*)`,`optional`): if specified, look for this (sub)string in the device name to filter available devices.

* `extensions` (`character(*)`,`optional`): if specified, look for these OpenCL extensions (command-separated) to filter available devices; any device that does not support all extensions specified will be filtered out.

* `sortBy` (one of `'core'`,`'memory'`,`'clock'`, `optional`): from the filtered list, choose the device with the most compute units or total memory or clock speed.

__Example:__
```fortran
type(fclDevice) :: device
...
device = fclInit(vendor='nvidia,amd',type='gpu',sortBy='cores')
```

In this example we have specified any `gpu` device belonging to vendors `nvidia` OR `amd` and to choose the device
with the most compute units (`cores`).

!!! note
    `fclInit` automatically creates an OpenCL context for the chosen device and uses it to set the __default context__.
    By setting the default context, subsequent Focal API calls can omit the context (`'ctx'`) argument.

To add more devices from the same vendor, see `fclFindDevices` below.

__[Jump down to command queues](#2-creating-a-command-queue) to now create a command queue on your chosen device__



## 1b. Advanced setup


### Querying platforms
OpenCL is able to support multiple different implementations on the same host using a platform model.
An OpenCL platform is a specific OpenCL implementation; in general, platforms coincide with different hardware vendors.
For example, if your machine has an Intel CPU and an NVIDIA graphics card both with drivers supporting OpenCL,
then you will have two platforms available: one each for Intel and NVIDIA.

We can get a list of platforms using the Focal subroutine `fclGetPlatforms()`.
This returns a list of Focal `fclPlatform` objects:

```fortran
type(fclPlatform), pointer :: platforms(:)

platforms => fclGetPlatforms()
```

The Focal platform object contains fields such as `name`, `vendor`, `version`, and `numDevice` which we can use
to select a particular platform.

__API ref:__
[fclPlatform](https://lkedward.github.io/focal/type/fclplatform.html),
[fclGetPlatforms](https://lkedward.github.io/focal/interface/fclgetplatforms.html),
[fclGetPlatformInfo](https://lkedward.github.io/focal/interface/fclgetplatforminfo.html)


### Create a context
We can explicitly create an OpenCL context with a particular platform or vendor using `fclCreateContext`:

There are two ways of calling this function, either with a platform object (see `fclGetPlatforms()` above to query platforms) or with 
a vendor string to specify the desired vendor:

__Interface:__
```fortran
ctx = fclCreateContext(platform)
ctx = fclCreateContext(vendor)
```

* `ctx` (`type(fclContext)`): context object returned.

* `platform` (`type(fclPlatform)`): a Focal platform object on which to create the context. 

* `vendor` (`character(*)`): string or comma-separate list of strings to select a particular device vendor. If multiple vendors are specified, then the first vendor 
in the list that is found on the system is chosen, *i.e.* specify vendors in order of preference.


__Example:__
```fortran
type(fclContext) :: ctx
...
ctx = fclCreateContext('nvidia,intel')
```

In this example we create a context with first preference `'nvidia'` and second preference `'intel'` as device vendors.


### The default context

Once created, the resulting context object (called `ctx` above) is used in subsequent Focal calls to indicate which context to use.
If you are only using one context throughout your program, then your code can be simplified by setting the *default context*.
This is a global variable which allows Focal calls to omit the context variable.

In the following examples, we query devices in a context for GPUs; the first example uses an explicit context variable `ctx` whereas the
second example calls `fclSetDefaultContext` and is able to omit the context variable in the call to `fclFindDevices`.

__Example: with explicit context variable__

```fortran
type(fclContext) :: ctx
type(fclDevice), allocatable :: devices(:)
...
ctx = fclCreateContext(vendor='nvidia')
devices = fclFindDevices(ctx,type='gpu')
```

__Example: with default context__

```fortran
type(fclDevice), allocatable :: devices(:)
...
call fclSetDefaultContext( fclCreateContext(vendor='nvidia') )
devices = fclFindDevices(type='gpu')
```


__API ref:__
[fclPlatform](https://lkedward.github.io/focal/type/fclplatform.html),
[fclSetDefaultContext](https://lkedward.github.io/focal/interface/fclsetdefaultcontext.html),
[fclDefaultCtx](https://lkedward.github.io/focal/module/focal.html#variable-fcldefaultctx)

### Query devices on the context

A useful way of querying available devices in a context is provided by the `fclFindDevices` function which enables us
to filter the device list based on device type, device name as well as sort the list according to device properties.

__Interface:__
```fortran
devices = fclFindDevices(ctx,[type],[nameLike],[extensions],[sortBy])
devices = fclFindDevices([type],[nameLike],[extensions],[sortBy])
```

where arguments `type`, `nameLike`, `extensions`, and `sortBy` have the same definitions as defined for `fclInit` above.

* `devices` (`type(fclDevice)`, `allocatable`): an array of devices allocated on assignment.

* `ctx` (`type(fclContext)`, `optional`): the context from which to query available devices. The __default context__ is used if this argument is omitted.

__Example:__
List CPUs in context `ctx` sorted (descending) by number of cores:

```fortran
type(fclContext) :: ctx
type(fclDevice), allocatable :: devices(:)
...
devices = fclFindDevices(ctx,type='cpu',sortBy='cores')
```

__Example:__
List GPUs in the default context where the name contains 'tesla' (case insensitive):

```fortran
type(fcLDevice), allocatable :: devices(:)
...
devices = fclFindDevices(type='gpu',nameLike='tesla')
```

From this list we can choose the first one or more devices as required.

We can further query device properties using `fclGetDeviceInfo` (this requires inclusion of the `clfortran` module which defines values for the property `key` argument).


__API ref:__
[fclFindDevices](https://lkedward.github.io/focal/interface/fclfinddevices.html),
[fclGetDeviceInfo](https://lkedward.github.io/focal/interface/fclgetdeviceinfo.html)
[fclDevice](https://lkedward.github.io/focal/type/fcldevice.html)


## 2. Creating a command queue
Once a context created, and a device selected (see above) we can create an OpenCL command queue; all device actions are submitted via command queues.
Command queues are associated with individual devices where one device can have multiple command queues (but not vice versa).
Command queues are created using the `fclCreateCommandQ` function.

__Interface__

```fortran
cmdq = fclCreateCommandQ(ctx,device,[enableProfiling],[outOfOrderExec], &
                           [blockingRead], [blockingWrite])
cmdq = fclCreateCommandQ(device,[enableProfiling],[outOfOrderExec], &
                           [blockingRead], [blockingWrite])
```

* `cmdq` (`type(fclCommandQ)`): the created command queue object

* `ctx` (`type(fclContext)`,`optional`), the context associated with `device`. If no context is specified, then the default context is used.

* `device` (`type(fclDevice)`): the device on which to create the command queue.

* `enableProfiling` (`logical`,`optional`): whether to enable event profiling on this command queue, default `.false.`

* `outOfOrderExec` (`logical`,`optional`): whether to enable out-of-order execution on this command queue, default `.false.`

* `blockingRead` (`logical`,`optional`): whether memory read operations are host-blocking on this command queue, default `.true.`

* `blockingWRite` (`logical`,`optional`): whether memory write operations are host-blocking on this command queue, default `.true.`

__Example:__
Create command queue on first device in `devices` list with profiling enabled.

```fortran
type(fclCommandQ) :: cmdq
...
cmdq = fclCreateCommandQ(devices(1),enableProfiling=.true.)
```

__API ref:__
[fclCreateCommandQ](https://lkedward.github.io/focal/interface/fclcreatecommandq.html),
[fclCommandQ](https://lkedward.github.io/focal/type/fclcommandq.html)


### 2.1 The default command queue

Once created, the resulting command queue object (called `cmdq` above) is used in subsequent Focal calls to indicate which device to use.
In a similar way to the default context, a *default command queue* is provided to allow subsequent focal calls to omit the command queue object.
This is useful if you are only using one command queue throughout your program.

__Example__
Set the default command queue on first device in `devices` list.
```fortran
call fclSetDefaultCommandQ( fclCreateCommandQ(devices(1)) )
```

__API ref:__
[fclSetDefaultCommandQ](https://lkedward.github.io/focal/interface/fclsetdefaultcommandq.html),
[fclDefaultCommandQ](https://lkedward.github.io/focal/module/focal.html#variable-fcldefaultcmdq)


### 2.2 Command queue pools

Many devices support multiple harware queues which allow different kernel and memory operations to be processed
concurrently; this is particularly important when wanting to overlap memory transfers with compute operations or when individual
compute kernels do not fully utilise the device compute units.

OpenCL allows multiple command queues to be created for a single device; if the device supports multiple hardware queues, then the OpenCL 
command queues will map in some way to the available hardware queues. If not supported then the command queue work will be serialised on the device

!!! note
    You can use the [tracing](../profiling/#14-generate-a-chrome-tracing-file) functionality to visually check for kernel/memory runtime concurrency.

Focal provides a `fclCommandQPool` type which performs simple round-robin scheduling across multiple command queues.
To create a command queue pool we can use the `fclCreateCommandQPool` command which accepts the same arguments are `fcCreateCommandQ` in 
addition to an argument `N` specifying the number of command queues to create.

__Interface__

```fortran
cmdq = fclCreateCommandQPool(ctx,N,device,[enableProfiling],[outOfOrderExec], &
                           [blockingRead], [blockingWrite])
cmdq = fclCreateCommandQPool(N,device,[enableProfiling],[outOfOrderExec], &
     
```

* `N` (`integer`): number of command queues to create within command queue pool.

Once created we can use the methods `next()` and `current()` to return the next scheduled or currently selected command queues respectively.

__Example:__

```fortran
type(fclCommandQPool) :: qPool
...
qPool = fclCreateCommandQPool(3,device)

do i=1,3
  kernel1%launch(qPool%next(),data(i))
  kernel2%launch(qPool%current(),data(i))
end do
```

In this example we launch two sequential kernels three times on three different command queues for three different 
sets of data. Note that the second kernel launches on the same queue as the first kernel and will hence be launched in
sequence, but each iteration of the do loop increments the current queue using the `next()` method.

__API ref:__
[fclCreateCommandQPool](https://lkedward.github.io/focal/interface/fclcreatecommandqpool.html),
[fclCommandQPool](https://lkedward.github.io/focal/type/fclcommandqpool.html)