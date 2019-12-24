# Profiling OpenCL code in Focal

OpenCL has a built in event profiling functionality whereby timings (SUBMIT, START, END) are available for enqueued operations.
This functionality can be handled automatically in Focal by enabling profiling on kernel and buffer objects.
When enabled, kernel launch events and buffer transfer events are 'recorded' for which timing information can be
later extracted and processed.

## 1. Enable profiling

Enabling profiling requires allocating space to 'record' events.
You can allocate space for the expected number of events or for a number less than this.
If less space is allocated than events enqueued, then only the last `N` events are recorded.
*i.e* event recording 'loops around' to overwrite previous events.

### 1.1 Kernels

When creating a kernel object, specify a `profileSize` for the number of kernel events to record.

__Example__

```fortran
type(fclProgram) :: prog
type(fclKernel) :: myKernel
...
myKernel = fclGetProgramKernel(prog,kernelName='sum',profileSize=100)
```

Here we extract a focal kernel object for the kernel `sum` with space for recording 100 events.

__API ref:__
[fclGetProgramKernel](https://lkedward.github.io/focal-api/interface/fclgetprogramkernel.html)

### 1.2 Buffers

When initialising a buffer object, specify a `profileSize` and optionally a `profileName` for the number of
buffer transfer events to record and a description for profiling output respectively.

__Example__

```fortran
integer :: N
type(fclFloat) :: deviceArray
...
deviceArray = fclBufferFloat(N,.true.,.false., &
                        profileSize=100,profileName='deviceArray')
```

Here we define a read-only (for kernels) float buffer of `N` elements with space to
record 100 transfer events.
We additionally specify the `profileName` using the variable name.

__API ref:__
[fclBufferInt32](https://lkedward.github.io/focal-api/interface/fclbufferint32.html),
[fclBufferFloat](https://lkedward.github.io/focal-api/interface/fclbufferfloat.html),
[fclBufferDouble](https://lkedward.github.io/focal-api/interface/fclbufferdouble.html)

## 2. Get profiling data

### 2.1 Print out profiling data

Once your OpenCL program has completed and all events are finished (see [host synchronisation](../events#2-host-synchronisation)),
we can use the command `fclDumpProfileData` to extract and display profiling results.

__Interfaces__

```fortran
call fclDumpProfileData([outputUnit=<integer>],kernelList=<fclKernel(:)>,device=<fclDevice>)
call fclDumpProfileData([outputUnit=<integer>],bufferList1=<fclBuffer(:)>,[bufferList2=<fclBuffer(:)>],[bufferList3=<fclBuffer(:)>])
```

- `outputUnit` (*optional*) specifies the output (file) unit to write profiling information.
If omitted then `stdout` is used.

- `kernelList` specifies a list of focal kernel objects for which to print out profiling data

- `device` is the device object on which the kernels were executed

- `bufferList` specifies a list of focal buffer objects for which to print out profiling data.
Use `bufferList1`, `bufferList2`, `bufferList3` for buffers of different types.

__Example__

```fortran
type(fclDevice) :: myDevice
type(fclKernel) :: initK, collideK, boundariesK, streamK, calcVarsK
type(fclDeviceFloat) :: velocities_d,rho_d,u_d,v_d
type(fclDeviceInt32) :: bcFlags_d
...
call fclDumpProfileData([initK,collideK,boundariesK,streamK,calcVarsK],myDevice)
call fclDumpProfileData([velocities_d,rho_d,u_d,v_d],[bcFlags_d])
```

This prints the following output to standard output:

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


### 2.2 Extract profiling data

To access profiling data directly, use the command `fclGetEventDurations` and pass in the `profileEvents` component
of the kernel or buffer object.
This will return an integer array where each entry corresponds to the duration in nanoseconds of an event in the input array.

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
