What is an operating system?
What happens during a context switch?
How memory management works?
Pinning - this doenst work coz the address might change.
TLB tagging.

# Class 2

Kernel vs User mode
System Calls
Interrupts and signals.

Signals are a way OS wants to talk to software and interrupts are a way hardware does with cpu.

System call causes a trap and the cpu switches to kernel mode. and then the instructions are fetched and executed.

There is a cost associated to system calls, coz we have to fetch the registers, instructions everything. 

L3 design : implements only 3 things in the micorkernel. address space, threads and IPC and unique identifiers.
