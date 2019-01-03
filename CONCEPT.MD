## Concepts (outdated)

### Slots

The kernel basically divides the 64-bit virtual memory into 'slots', each slot is 4GB in size. Assuming a 47-bit 
addressing scheme, we can just consider 128TB of the address space and ignore the rest. In that case the 
memory can have 128T/4G=32K slots.

When you load a new program to memory (like memman - user-space memory manager), the microkernel reserves a new 4GB 
slot in the 64-bit virtual address space. The microkernel includes a platform-specific loader that parses
the memman binary file and loads it to the virtual address space. Thus, you can imagine slots as separate
4GB modules that map to programs that are currently loaded in memory.

Slot 0 is a 1-to-1 mapping for the lower 4GB of physical address space. The slot is used to access kernel code and
data. Slot 1 is used when the first program (init) is loaded to memory on initialization. The "init" program
loads other programs to memory: memman, devman, etc. Each program is assigned the next available slot.
Typically, all platform-specific code goes inside the kernel; while user programs (like memman, devman, etc.)
should be platform-agnostic. The "init" program may need to be aware of the machine itself (is it an IBM PC or Macintosh?)
to load proper programs on initialization.

### Contexts

A context is a representation for the architectural state of the CPU at a point of time. In x86, the context
contains also the state of the GDT. Usually, segment selectors 0x08 and 0x10 are the code and data segment
selectors for kernel slot. i.e., their base should be 0x00000000LL and the limit should be 0x100000000LL.
segment selectors 0x18 and 0x20 are the code and data segment selectors for the slot that is currently
being executed by this specific context. Their limit should be 4GB as well.

Each context has its own GDT. For example, context A might have segment descriptors 0x18/0x20 referring
to slot 1, while the same segment descriptors of context B are referring to slot 2. This means that
context A is currently executing slot 1 code, while context 2 is executing slot 2. Since all slots
(except kernel) include only user-space code, they cannot modify GDT; therefore they cannot access 
other slots' code/data. A context whose CS register is equal to 0x08 instead of 0x18 is currently 
processing a kernel system call, and it will return soon to 0x18.

That being said, the system can have several contexts at the same time. More than one context
can be referring to the same slot! This is equivalent to "kernel threads" in the UNIX operating system.
A slot might not be referred to by any context (example: a server that is not processing any requests
at the moment). Contexts can be active, waiting (ready), and/or blocked. The kernel maintains a queue
of contexts and uses a round-robin mechanism to loop over contexts.

### Loader

Besides platform-specific routines, context switching routines, and slot management, the microkernel
also contains a platform-specific loader to parse binary files and load them to slots. The loader
calls a user-space filesystem driver to read the binary file and load it to memory when requested.
A request to load binary file into a new slot can be initiated by user-space programs.

This is kind of different from the UNIX system. You don't need a virtual memory fork to 
load a new program to memory. But you basically need to create a new context. But first,
You need to ask the kernel to load a binary file to a new slot.

For example, assume devman is loaded to slot 10, and it wants to load a PCI driver to memory.
For simplicity, assume that the next available slot is 11. First, devman should ask the microkernel's
loader to load the PCI driver to a new slot. The kernel handles the whole loading process and returns
11, the number of the loaded slot.

Next, devman (slot 10) would create a new context. This is similar to UNIX' fork. The new context
will have its own stack, but it is still executing slot 10 code. The code of the new context
should initiate an IPC request and send a message to slot 11. The operation code field in
the IPC message should be 0, which means "start" command. That way slot 11 starts execution.

### Message Passing

A context executing in slot X can send messages to slot Y and messages can be processed without
context switching, thanks to the concepts introduced above. The microkernel provides
an interface to send messages to anonymous slots; and each slot should provide a message
handler.

When a given context initiates a message request, this request itself is a system call that
switches CS/DS to segment selectors 0x08/0x10. The current RIP value is stored in
the user stack associated with that context. If the requested is initiated by
context running in slot X, GDT's 0x18/0x20 are referring to the memory of slot X by default.
The kernel modifies these two segment descriptors to refer to slot Y instead.

Next, the kernel maps the context's stack to slot Y. When the kernel code returns to user
space (to continue the execution of the context), it returns to slot Y's message handler
instead of slot X's code. When slot Y has done processing the message, it calls
a specific system call where the kernel unmaps the context stack from the memory
of slot Y, then resets GDT's 0x18/0x20 segments to slot X, and returns to slot X again.

As you can see, a context is not tied to a specific slot, but instead it flies from
one slot to another. The slot might start in devman, then fly to procman, then
return to devman again, then fly to memman, then fly to fsman, then to devman (this
is not a return to devman, but actually a new call), and so on, until it is killed.

This "fly" operation is actually the cool `ipc_call()` system call. Messages are always
synchronous: The context will not return to the original calling slot until
`ipc_return()` is called. Asynchronous message passing can be easily implemented
over that. 

In all cases, no virtual memory switching is needed, because all slots share
the same 64-bit virtual memory. We just need to modify GDT to ensure protection. That
being said, The `ipc_call()` system call is not expensive at all, compared to
other mainstream microkernel implementations.

### Initialization

On UEFI systems, no bootloader is needed. The kernel can be compiled as an EFI stub (just
like Linux kernel) and directly loaded to memory by firmware. The kernel next
takes control.

The initialization code should initialize x86 registers, create page directory
and page tables, and creates an initial context representing the current
execution track.

The loader then loads the "init" program to memory. But how can it load a program
from disk while file-system manager is not loaded at all? Therefore a UEFI
interface component is implemented inside the microkernel. The component
handles all tty print() operations and all disk read() operations until
the appropriate programs are loaded to memory.

### The microkernel

To conclude, the microkernel contains the following components:

* `init.c`: initialization sequence.
* `fw.h`: firmware interface; implemented by `fw_efi.c`.
* `tty.c`: printk interface (initially handled by fw).
* `file.c`: fread interface (initially handled by fw).
* `load.h`: platform-specific loader; implemented by `load_elf.c`.
* `splash.c`: prints splash screen using printk.
* `arch.h`: architecture-related routines; implemented by `arch_x86.c`.
* `con.c`: context manager and switching.
* `slot.c`: slot manager.
* `ipc.c`: message-passing manager.
* `syscall.c`: system-call interface.

The `init()` function does the following:
1. `fw_init() # handled by efi_init()`
2. `splash_init()`
3. `arch_init() # handled by x86_init()`
4. `con_init()`
5. `slot_init()`
6. `ipc_init()`

`slot_init()` loads slot 1 with "init" program, and `ipc_init()` calls `ipc_call()` to
initiate a "start" command to the "init" program.
