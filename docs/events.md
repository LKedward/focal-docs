# Events and synchronisation

Most OpenCL commands involve enqueueing an operation to a command queue and returning control back to the host program.
If an in-order command queue is used then these operations are guaranteed to be executed in the order that they were submitted.

Implicit host-device synchronisations can occur if blocking data transfers are used (see [host-device transfers](../memory/#3-data-transfer-between-device-and-host)).
To explicitly keep track of enqueued operations on the host, we can use event objects issued with each command.

If an out-of-order command queue is used, then additional consideration must be made for inter-dependencies between the operations enqueued;
this is enabled by command queue barriers and prerequisite events.

## 1. Events and host synchronisation

### 1.1 Event objects

The following events are available following certain enqueing operations:

| Action                               | Event Object                                   | Action Example                   |
|--------------------------------------|------------------------------------------------|----------------------------------|
| Transfer host array to device buffer | `fclLastWriteEvent`,<br>`cmdq%lastWriteEvent`  | `fortrandeviceArray = hostArray` |
| Transfer device buffer to host array | `fclLastReadEvent`<br>`cmdq%lastReadEvent`     | `hostArray = deviceArray`        |
| Copy device buffer to device buffer  | `fclLastCopyEvent`<br>`cmdq%lastCopyEvent`     | `deviceArray2 = deviceArray1`    |
| Launch kernel                        | `fclLastKernelEvent`<br>`cmdq%lastKernelEvent` | `myKernel%launch()`              |

__API ref:__
[fclCommandQ](https://lkedward.github.io/focal-api/type/fclcommandq.html),
[fclLastWriteEvent, fclLastReadEvent, fclLastCopyEvent, fclLastKernelEvent](https://lkedward.github.io/focal-api/module/focal.html#variable-fcllastwriteevent)

### 1.2 Waiting on the host

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
[fclWait](https://lkedward.github.io/focal-api/interface/fclwait.html)




## 3. Out-of-order execution and barriers

If a device supports it, then a command queue can be created with __out-of-order execution__ enabled which allows it to dynamically schedule enqueued kernels for maximum device utilisation.
This means that kernels and enqueued data transfers are not guaranteed to start and finish execution in the order that they were enqueued.
To control kernel and data dependencies in an out-of-order command queue, two mechanisms are available: barriers and prerequisite events.

### 3.1 Command queue barriers

OpenCL barriers break up the command queue into regions between which there can be no re-orderig of operations.
If a series of operations (group A) is followed by a barrier then followed by another series of operations (group A), then all events from group a must be complete (in any order) before any event in group B can start.

To enqueue a barrier in Focal, use `fclBarrier`:

__Interfaces__

```fortran
call fclBarrier()
call fclBarrier(<fclCommandQ>)
```

When called with no arguments, this enqueues a barrier onto the __default command queue__.
Otherwise, it enqueues a barrier onto the command queue specified in the first argument.

__API ref:__
[fclBarrier](https://lkedward.github.io/focal-api/interface/fclbarrier.html)


### 3.2 Prerequisite events

Prerequisite events are set prior to enqueueing an operation to specify events that must first complete for the following operation to start.

*Implementation under development*



