# Events and synchronisation

Most OpenCL commands involve enqueueing an operation to a command queue and returning control back to the host program.
If an in-order command queue is used then these operations are guaranteed to be executed in the order that they were submitted.
Implicit host-device synchronisations can occur if blocking data transfers are used (see [host-device transfers](../memory/#3-data-transfer-between-device-and-host)).

However to keep track of when enqueued operations explicitly, we can use event objects issued with each command.

*Full documentation pending.*

__API ref:__
[fclWait](https://lkedward.github.io/focal-api/interface/fclwait.html),
[fclBarrier](https://lkedward.github.io/focal-api/interface/fclbarrier.html)