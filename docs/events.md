# Events and synchronisation

Most OpenCL commands involve enqueueing an operation to a command queue and returning control back to the host program.
If an in-order command queue is used then these operations are guaranteed to be executed in the order that they were submitted.

Implicit host-device synchronisations can occur if blocking data transfers are used (see [host-device transfers](../memory/#3-data-transfer-between-device-and-host)).
To explicitly keep track of enqueued operations on the host, we can use event objects issued with each command.

If an out-of-order command queue is used, then additional consideration must be made for inter-dependencies between the operations enqueued;
this is enabled by command queue barriers and dependencies.


## 1. Event objects

Event objects represent operations that have been submitted to a command queue and provide a way of keeping track and specifying dependencies.

The following event objects are made available following certain enqueing operations:

| Action                               | Event Object                                     | Action Example                   |
|--------------------------------------|--------------------------------------------------|----------------------------------|
| Transfer host array to device buffer | `fclLastWriteEvent`,<br>`cmdq%lastWriteEvent`    | `deviceArray = hostArray` |
| Transfer device buffer to host array | `fclLastReadEvent`<br>`cmdq%lastReadEvent`       | `hostArray = deviceArray`        |
| Copy device buffer to device buffer  | `fclLastCopyEvent`<br>`cmdq%lastCopyEvent`       | `deviceArray2 = deviceArray1`    |
| Launch kernel                        | `fclLastKernelEvent`<br>`cmdq%lastKernelEvent`   | `myKernel%launch()`              |
| Enqueue queue barrier                | `fclLastBarrierEvent`<br>`cmdq%lastBarrierEvent` | `call fclBarrier(...)`           |

__API ref:__
[fclCommandQ](https://lkedward.github.io/focal/type/fclcommandq.html),
[fclLastWriteEvent, fclLastReadEvent, fclLastCopyEvent, fclLastKernelEvent, fclLastBarrierEvent](https://lkedward.github.io/focal/module/focal.html#variable-fcllastwriteevent)





## 2. Host synchronisation

### 2.1 Waiting for events

To wait on the host for device events to complete, use the `fclWait` command.
This command has multiple interfaces defined below.

__Interfaces__

```fortran
call fclWait()
call fclWait(<fclCommandQ>)
call fclWait(<fclEvent>)
call fclWait(<fclEvent(n)>)
```

When called without any arguments `fclWait` will wait for all events on the __default command queue__ to finish.

Alternatively, a specific command queue may be specified by passing it as an argument to `fclWait`.

To wait for a specific event to finish, pass that event to `fclWait`.

Alternatively, to wait for multiple events to complete, pass an array of `fclEvent` objects to `fclWait`.

__Example:__
Wait on a specific command queue.

```fortran
type(fclCommandQ) :: myCommandQ
...
call fclWait(myCommandQ)
```

__Example:__
Wait for a kernel event to complete.

```fortran
type(fclKernel) :: myKernel
type(fclEvent) :: e
...
myKernel%launch()          ! Launch kernel
e = fclLastKernelEvent     ! Save kernel event object
...                        ! Perform other operations
call fclWait(e)            ! Now wait for kernel to complete
```

__Example:__
Enqueue multiple asynchronous data transfers.

```fortran
type(fclCommandQ) :: cmdq
type(fclDeviceFloat) :: a1, a2, a3
type(fclEvent), dimension(3) :: e
...
cmdq%blockingWrite = .false.   ! Enable asynchronous host-to-device transfers
...                             
a1 = hostArray1                ! Enqueue first transfer
e(1) = cmdq%lastWriteEvent     ! Save first transfer event
a2 = hostArray2                ! Enqueue second transfer
e(2) = cmdq%lastWriteEvent     ! Save second transfer event
a3 = hostArray3                ! Enqueue third transfer
e(3) = cmdq%lastWriteEvent     ! Save third transfer event
...                            ! Perform other operations
call fclWait(e)                ! Now wait for transfers to complete
```

__API ref:__
[fclWait](https://lkedward.github.io/focal/interface/fclwait.html)



## 3. Dependencies and barriers

If using a single command queue with out-of-order execution disabled, then commands are guaranteed to be executed in the order that they were enqueued.
When using multiple command queues to submit work, or when using command queues with out-of-order execution enabled, then explicit management of dependencies is required
to ensure that work is performed in the correct order; two useful mechanisms are available for this: barriers and event dependencies.

### 3.1 Event dependencies

Event dependencies are set prior to enqueueing an operation to specify operations that must first complete for the following operation to start.
The Focal command `fclSetDependency` is used to specify dependencies for the next enqueued operation.

__Example:__
A kernel event depends on previous data transfers

```fortran
type(fclDeviceFloat) :: deviceArray1, deviceArray2
type(fclKernel) :: myKernel1, myKernel2
type(fclEvent) :: e(2)
type(fclCommandQ) :: cmdq2
...
call fclSetDefaultCommandQ( fclCreateCommandQ(devices(1),blockingWrite=.false.) )
cmdq2 = fclCreateCommandQ(devices(1))

deviceArray1 = hostArray
e(1) = fclLastWriteEvent         ! Save first write event
deviceArray2 = hostArray
e(2) = fclLastWriteEvent         ! Save second write event
...
! Launch kernel with no dependencies on separate cmdq
myKernel1%launch(cmdq2)
...
! Launch kernel with dependency on events in e
call fclSetDependency(e)         
myKernel2%launch(deviceArray1, deviceArray2)
```

In this example several host-to-device transfers are enqueued followed immediately by an independent kernel execution;
this first kernel can theoretically be executed at the same time as the transfers are occuring.
A second kernel is enqueued with explicit dependencies on the previous transfers; this kernel won't execute until
the transfers are complete.

!!! note
    Unless `hold=.true.` is specified in `fclSetDependency` then, dependencies are cleared after each enqueued operations.
    *i.e.* event dependencies will only apply to the next enqueued operation
	  not to subsequent ones. See example below for multiple events sharing the same dependency.

__Example:__
Apply the same event dependencies to multiple subsequent commands

```fortran
type(fclKernel) :: myKernel
type(fclEvent) :: e
...
myKernel%launch()
e = fclLastKernelEvent
...
call fclSetDependency(e,hold=.true.) ! Data transfers all depend on previous kernel launch
hostData1 = deviceData1
hostData2 = deviceData2
hostData3 = deviceData3
call fclClearDependencies()          ! Clear held dependencies

```


__API ref:__
[fclSetDependency](https://lkedward.github.io/focal/interface/fclsetdependency.html),
[fclClearDependencies](https://lkedward.github.io/focal/interface/fclcleardependencies.html)


### 3.2 Command queue barriers

OpenCL barriers break up the command queue into regions between which there can be no re-ordering of operations.
If a series of operations (group A) is followed by a barrier then followed by another series of operations (group A), then all events from group a must be complete (in any order) before any event in group B can start.

To enqueue a barrier in Focal, use `fclBarrier`:

__Interfaces__

```fortran
call fclBarrier()
call fclBarrier(<fclCommandQ>)
```

When called with no arguments, this enqueues a barrier onto the __default command queue__.
Otherwise, it enqueues a barrier onto the command queue specified in the first argument.

The barrier can be waiting upon or set as a dependency using the event objects: `fclLastBarrierEvent` or `cmdq%lastBarrierEvent`;
this is useful in referencing groups of events.

__Example:__
Wait on a group of asynchronous transfers to complete

```fortran
type(fclCommandQ) :: cmdq
type(fclDeviceFloat) :: a, b, c
...
cmdq = fclCreateCommandQ(devices(1),blockingWrite=.false.)
...
! Enqueue asynchronous transfers to device
a = hostA   
b = hostB
c = hostC

! Enqueue barrier
call fclBarrier(cmdq)
...
! Wait for all transfers to complete
call fclWait(cmdq%lastBarrierEvent)
...
! OR set a dependency for all transfers to complete
call fclSetDependency(cmdq%fclLastBarrierEvent)
...
```

__API ref:__
[fclBarrier](https://lkedward.github.io/focal/interface/fclbarrier.html),
[fclWait](https://lkedward.github.io/focal/interface/fclwait.html),
[fclSetDependency](https://lkedward.github.io/focal/interface/fclsetdependency.html),
[fclCommandQ](https://lkedward.github.io/focal/type/fclcommandq.html),
[fclLastBarrierEvent](https://lkedward.github.io/focal/module/focal.html#variable-fcllastwriteevent)


### 3.3 User events

OpenCL user events provide a way for device operations to wait for the completion of host operations.
A user event is first created which can be used as a dependency for subsequently enqued device operations.
The host program can then trigger the user event to signal its completion and allow it dependents to start executing.

__Example:__

```fortran
type(fclEvent) :: myHostEvent
type(fclKernel) :: myKernel
...
! Create user event
myHostEvent = fclCreateUserEvent()

! Set dependency on user event
call fclSetDependency(myHostEvent)
call myKernel%launch()
... ! Do some other work

! Trigger user event to signal completion
call fclSetUserEvent(myHostEvent)
! Kernel will now run
```

__API ref:__
[fclCreateUserEvent](https://lkedward.github.io/focal/interface/fclcreateuserevent.html),
[fclSetUserEvent](https://lkedward.github.io/focal/interface/fclsetuserevent.html)