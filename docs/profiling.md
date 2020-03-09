# Profiling OpenCL code in Focal

OpenCL has a built in event profiling functionality whereby timings are available for enqueued operations.
This functionality can be handled automatically in Focal by enabling profiling on kernel and buffer objects.
When enabled, kernel launch events and buffer transfer events are 'recorded' for which timing information can be
later extracted and processed.

__Example program:__ [nbody.f90](https://github.com/LKedward/focal/blob/master/examples/nbody.f90)

## 1. Standard: Use a profiler object

An `fclProfiler` type is provided to simplify the extraction of profiling data.
This type simply represents a collection of kernels and buffers for which you wish to
generate profiling data.
A set of routines are provided to extract and present profiling data for all kernels
and buffers in a particular profiler object.

### 1.1 Define a profiler

To start, we must define a profiler object and set the device it is associated with:

Assuming we have a context `ctx` and we have a device list `devices` (see [Setup: Querying devices](../setup/#4-querying-devices))
from which we are going to use the first device, then we set the profiler device as follows:

```fortran
type(fclProfiler) :: profiler
type(fclDevice), allocatable :: devices(:)
...
devices = fclFindDevices(ctx,type='cpu',sortBy='cores')
profiler%device = devices(1)
```

__API ref:__
[fclProfiler](https://lkedward.github.io/focal-api/type/fclprofiler.html)

### 1.2 Enable profiling on kernels and buffers

Once we have a profiler with an associated device and have initialised our
kernels and buffers, we can enable profiling with the following interfaces:

__Interface:__

```fortran
call fclProfilerAdd(<fclProfiler>,profileSize=<int>,<fclKernel|fclBuffer>,...)
```

or equivalently:

```fortran
call <fclProfiler>%add(profileSize=<int>,<fclKernel|fclBuffer>,...)
```

Both interface formats are equivalent.

- `profileSize` specifies the amount of space to allocate for recording events:
set to the number of events you wish to record.
If more events occur than space allocated, then 'old' events are simply overridden.
*i.e.* only the last `profileSize` events are recorded.

- Up to 10 kernel or buffer objects can be passed as arguments after `profileSize`.
The same profile size is allocated for each kernel or buffer specified.

__Example:__

```fortran
type(fclProfiler) :: profiler
type(fclKernel) :: myKernel1, myKernel2
type(fclDeviceInt32) :: buffer1
type(fclDeviceInt32) :: buffer2
...
profiler%add(100,myKernel1,myKernel2)
profiler%add(10,buffer1,buffer2)
```

In this example, we enable profiling for both kernels with space for 100 events and,
in a separate call, we enable profiling for two buffer objects with space for 10 events.

!!! note
    If intending to profile a buffer object, make sure you specify a `profileName`
    when [initialising](../memory/#1-initialise-device-memory) the buffer so that
    the profiler can print it in the output.


__API ref:__
[fclProfilerAdd](https://lkedward.github.io/focal-api/interface/fclprofileradd.html)


### 1.3 Print out profiling data summary

Once your OpenCL program has completed and all events are finished (see [host synchronisation](../events#2-host-synchronisation)),
we can use the command `fclDumpProfileData` with our profiler object to extract and display profiling results.

__Interface:__

```fortran
call fclDumpProfileData(<fclProfiler>,outputUnit=<int>)
```

- The first argument is the profiler object for which we wish to print out data for.

- `outputUnit` is an __optional__ integer argument to specify an open file unit to which to print
profiling data summary. If omitted then `stdout` is used and data is printed to the screen.

__Example:__

```fortran
type(fclProfiler) :: profiler
...
call fclWait()
call fclDumpProfileData(profiler)
```

The following is an example of output from `fclDumpProfileData`:

```
 -----------------------------------------------------------------------------
        Profile name|  No. of|   T_avg|   T_max|   T_min| Local|Privat|PWGS
            (Kernel)|  events|    (ns)|    (ns)|    (ns)|  Mem.|  Mem.|
 -----------------------------------------------------------------------------
          initialise|       1|  308750|  308750|  308750|     0|     0|  32
             collide|    5000|  846595| 1527166|  735833|     0|     0|  32
  boundaryConditions|    5000|  154161|  234083|  131333|     0|     0|  32
              stream|    5000| 1067590| 1419500|  921000|     0|     0|  32
           macroVars|      50|  897174| 1104250|  802833|     0|     0|  32
 -----------------------------------------------------------------------------
  ns: nanoseconds,  PWGS: Preferred work group size,  Mem: Memory in bytes.
 -----------------------------------------------------------------------------


 -----------------------------------------------------------------------------
        Profile name|  No. of|    Size|Transfer|   S_avg|   S_max|   S_min
            (Buffer)|  events|(KBytes)|    mode|  (GB/S)|  (GB/S)|  (GB/S)
 -----------------------------------------------------------------------------
               rho_d|      50|     640|   READ | 15.2255| 20.7570|  6.9565
                 u_d|      50|     640|   READ | 14.3151| 18.7777|  7.3074
                 v_d|      50|     640|   READ | 11.1791| 15.3909|  6.0425
           bcFlags_d|       1|     640|   WRITE|  9.3842|  9.3842|  9.3842
 -----------------------------------------------------------------------------
```


__API ref:__
[fclDumpProfileData](https://lkedward.github.io/focal-api/interface/fcldumpprofiledata.html)


### 1.4 Generate a chrome tracing file

The Google Chrome web browser has a built-in profiler which can
[load and display](https://aras-p.info/blog/2017/01/23/Chrome-Tracing-as-Profiler-Frontend/)
any arbitrary profiling data as long as it is in the expected
[JSON format](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview).

The `fclDumpTracingData` routine is provided to generate such JSON files for use
with chrome tracing.

__Interface:__

```fortran
call fclDumpTracingData(<fclProfiler>,filename=<character(*)>)
```

__Example:__

```fortran
type(fclProfiler) :: profiler
...
call fclWait()
call fclDumpTracingData(profiler,'trace.json')
```

__API ref:__
[fclDumpTracingData](https://lkedward.github.io/focal-api/interface/fcldumptracingdata.html)

## 2. Advanced: Get timings explicitly

### 2.1 Enable profiling without a profiler object

The `fclProfiler` type provides a convenient interface for producing aggregate
profiling information for a collection of kernels and buffers.
However a profiler object is not required if you want to access profiling data directly.
To enable profiling on a kernel or buffer directly, use `fclEnableProfiling`.

__Interfaces:__

```fortran
call fclEnableProfiling(<fclKernel>,profileSize=<int>)
call fclEnableProfiling(<fclBuffer>,profileSize=<int>)
```


__Example:__

```fortran
type(fclKernel) :: myKernel
type(fclDeviceFloat) :: myBuffer
...
call fclEnableProfiling(myKernel,100)
call fclEnableProfiling(myBuffer,50)
```

__API ref:__
[fclEnableProfiling](https://lkedward.github.io/focal-api/interface/fclenableprofiling.html)


### 2.2 Extract event durations

To access profiling data directly, use the command `fclGetEventDurations` and pass in the `profileEvents` component
of the kernel or buffer object.
This will return an integer array where each entry corresponds to the duration in __nanoseconds__ of an event in the input array.

__Interface__

```fortran
<integer(:)> = fclGetEventDurations(eventList=<fclEvent(:)>)
```

__Example__

```fortran
integer, parameter :: Nprofile
type(fclKernel) :: myKernel
integer :: durations(Nprofile)
real :: averageTime
...
myKernel = fclGetProgramKernel(prog,kernelName='sum',profileSize=Nprofile)
...
durations = fclGetEventDurations(myKernel%profileEvents(:))
averageTime = sum(durations)/Nprofile
```


__API ref:__
[fclGetEventDurations](https://lkedward.github.io/focal-api/interface/fclgeteventdurations.html)
