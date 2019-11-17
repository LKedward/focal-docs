# Setup

All OpenCL actions occur via contexts, which are containers for related devices, and command queues which are attached to specific devices.
Each context is associated with a specific platform where a platform generally coincides with a particular vendor.
Before we can create a context, we need to see which platforms are available.

Skip to [here](#22-with-a-vendor-string) for quickly creating a context just by specifying the vendor name.

## 1. Querying platforms
OpenCL is able to support multiple different implementations on the same host using a platform model.
An OpenCL platform is a specific OpenCL implementation; in general, platforms coincide with different hardware vendors.
For example, if your machine has an Intel CPU and an NVIDIA graphics card both with drivers supporting OpenCl,
then you will have two platforms available: on each for Intel and NVIDIA.

We can get a list of platforms using the Focal subroutine `fclGetPlatforms()`.
This returns a list of Focal `fclPlatform` objects:

```fortran
type(fclPlatform), pointer :: platforms(:)

platforms => fclGetPlatforms()
```

The Focal platform object contains fields such as `name`, `vendor`, `version`, and `numDevice` which we can use
to select a particular platform.

__API ref:__
[fclPlatform](https://lkedward.github.io/focal-api/type/fclplatform.html), 
[fclGetPlatforms](https://lkedward.github.io/focal-api/interface/fclgetplatforms.html),
[fclGetPlatformInfo](https://lkedward.github.io/focal-api/interface/fclgetplatforminfo.html)


## 2. Creating a context

### 2.1 With a platform object
Having queried available platforms as above, we can create an Focal context object using `fclCreateContext`:

```fortran
type(fclPlatform), pointer :: platforms(:)
type(fclContext) :: ctx
...
platforms => fclGetPlatforms()
ctx = fclCreateContext(platforms(i))
```

Here we have chosen the `i`*th* platform from the `platforms` array.

__API ref:__
[fclCreateContext](https://lkedward.github.io/focal-api/interface/fclcreatecontext.html)


### 2.2 With a vendor string
It is commonly required to choose a platform based on the vendor, so an alternative syntax for creating a context
is provided whereby a string is used to find a matching platform vendor:

```fortran
type(fclContext) :: ctx
...
ctx = fclCreateContext(vendor='nvidia')
```

Here we have requested a context be created on the first platform where the vendor string contains 'nvidia' (case insensitive).

__API ref:__
[fclCreateContext](https://lkedward.github.io/focal-api/interface/fclcreatecontext.html)


### 2.3 The default context

Once created, the resulting context object (called `ctx` above) is used in subsequent Focal calls to indicate which context to use.
If you are only using one context throughout your program, then your code can be simplified by setting the *default context*.
This is a global variable which allows Focal calls to omit the context variable.

In the following examples, we query devices in a context for GPUs; the first example uses an explicit context variable `ctx` whereas the
second example calls `fclSetDefaultContext` and is able to omit the context variable in the call to `fclFindDevices`.

__Example: with explicit context variable__

```fortran
type(fclContext) :: ctx
type(fclDevice), pointer :: devices(:)
...
ctx = fclCreateContext(vendor='nvidia')
devices => fclFindDevices(ctx,type='gpu')
```

__Example: with default context__

```fortran
type(fclDevice), pointer :: devices(:)
...
call fclSetDefaultContext( fclCreateContext(vendor='nvidia') )
devices => fclFindDevices(type='gpu')
```


__API ref:__
[fclPlatform](https://lkedward.github.io/focal-api/type/fclplatform.html), 
[fclSetDefaultContext](https://lkedward.github.io/focal-api/interface/fclsetdefaultcontext.html), 
[fclDefaultCtx](https://lkedward.github.io/focal-api/module/focal.html#variable-fcldefaultctx)


## 4. Querying devices

A useful way of querying available devices is provided by the `fclFindDevices` function which enables us
to filter the device list based on device type, device name as well as sort the list according to device properties.

__Example:__
List CPUs in context `ctx` sorted (descending) by number of cores:

```fortran
type(fclContext) :: ctx
type(fclDevice), pointer :: devices(:)
...
devices => fclFindDevices(ctx,type='cpu',sortBy='cores')
```

__Example:__
List GPUs in the default context where the name contains 'tesla' (case insensitive):

```fortran
type(fcLDevice), pointer :: devices(:)
...
devices => fclFindDevices(type='gpu',nameLike='tesla')
```

From this list we can choose the first one or more devices as required.

We can further query device properties using `fclGetDeviceInfo` (this requires inclusion of the `clfortran` module which defines values for the property `key` argument).

__API ref:__
[fclFindDevices](https://lkedward.github.io/focal-api/interface/fclfinddevices.html),
[fclGetDeviceInfo](https://lkedward.github.io/focal-api/interface/fclgetdeviceinfo.html)
[fclDevice](https://lkedward.github.io/focal-api/type/fcldevice.html)


## 5. Creating a command queue
Once a context created, and a device selected (see above) we can create an OpenCL command queue; all device actions are submitted via command queues.
Command queues are associated with individual devices where one device can have multiple command queues (but not vice versa).
Command queues are created using the `fclCreateCommandQ` function.

__Interface__

```fortran
<fclCommandQ> = fclCreateCommandQ(<fclContext>,<fclDevice>,enableProfiling=<logical>,outOfOrderExec=<logical>)
<fclCommandQ> = fclCreateCommandQ(<fclDevice>,enableProfiling=<logical>,outOfOrderExec=<logical>)
```
If no context is specified, then the default context is used.
Arguments `enableProfiling` and `outOfOrderExec` are optional, both are disabled (`.false.`) by default;
 profiling is disabled and command queue execution is __in-order__.

__Example:__
Create command queue on first device in `devices` list.

```fortran
type(fclCommandQ) :: cmdq
...
cmdq = fclCreateCommandQ(devices(1))
```

__API ref:__
[fclCreateCommandQ](https://lkedward.github.io/focal-api/interface/fclcreatecommandq.html),
[fclCommandQ](https://lkedward.github.io/focal-api/type/fclcommandq.html)


### 5.1 The default command queue

Once created, the resulting command queue object (called `cmdq` above) is used in subsequent Focal calls to indicate which device to use.
In a similar way to the default context, a *default command queue* is provided to allow subsequent focal calls to omit the command queue object.
This is useful if you are only using one command queue throughout your program.

__Example__
Set the default command queue on first device in `devices` list.
```fortran
call fclSetDefaultCommandQ( fclCreateCommandQ(devices(1)) )
```

__API ref:__
[fclSetDefaultCommandQ](https://lkedward.github.io/focal-api/interface/fclsetdefaultcommandq.html),
[fclDefaultCommandQ](https://lkedward.github.io/focal-api/module/focal.html#variable-fcldefaultcmdq)