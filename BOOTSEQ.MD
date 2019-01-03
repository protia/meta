## Boot Sequence

In this file, we explain a proposed sequence for system initialization process.

### UEFI/BOOTx64.EFI Environment

As a start, UEFI provides an initial page table which is extended 
by the boot loader to load all start-up programs. The
boot loader contains Systema's "core loader" DLL file that
loads binary files into memory (any memory). UEFI
already provides a disk interface to load sectors
from disk. The boot loader also contains a file system
DLL file that parses file-systems. Thus, if Protia's
native file-system is not supported by MS-DOS, you
still can boot from a Protia-formatted disk.

This is the magic of clean and modularized software design;
the core loader DLL file doesn't depend on a specific
file-system or memory manager. You need to pass
FileSystemInterface and MemoryManagerInterface to the
loader, and it will call their member functions 
to load files from file system to memory. In this sense,
FileSystemInterface encapsulates the boot-loader
capability of parsing file-systems through its DLL component,
and MemoryManagerInterface encapsulates the
boot-loader ability to access and manage both
physical and virtual memories.

Similarly, the boot loader provides the core filesystem
DLL with DiskInterface, which encapsulates the boot-loader
capability of interfacing UEFI's disk interface! These
DLLs are provided by the "file-system" and "systema-loader"
packages, and the boot loader is statically linked against
them, eliminating the need for a pre-loader for the bootloader!

### What ServMan needs to know about the bootloader

ServMan doesn't need to make any assumptions on how the
bootloader works, so it doesn't need to depend on the
bootloader at all. It just starts execution like any other
program in Protia, with the guarantee that the environment
has already been initialized by the boot loader as we described 
above.

In fact, it is the boot-loader that depends on ServMan (not
the other way around): it just needs to know the data
structure needed to be passed to ServMan before ServMan
starts execution. This data structure contains a list of
all loaded start-up programs (loaded by the bootloader)
and other cool information for start-up. It is defined in
one of the include files installed by ServMan on the disk.

It is like you are calling the command "dd" on UNIX: you depend
on dd. You need to know how dd's command-line arguments
are structured before you execute it, but dd doesn't need
to know you. Indeed, it doesn't give a shit.

The same analogy is applied here: the bootloader is the one
calling ServMan, not the other way around. It is the one
that assumes that ServMan is installed to the system.
That's why the dependency is initiated by the bootloader,
and ServMan doesn't depend on anything except a basic
DataStructs package providing basic String operations
and basic data structures (LinkedList, Stack, Queue...).

### Summary of the steps

So, we can summarize the boot sequence in these steps:

  1. A great page table is created in physical memory by UEFI,
     this page table provides mapping from virtual address
     space to physical memory

  2. UEFI loads BOOTx64.EFI an MS-DOS partition.
  
  3. Depedning on how BOOTx64.EFI is configured, it looks
     for the partition that contains Protia operating system.
  
  4. BOOTx64.EFI looks for C:\Protia\Config.sys file.
     It contains a list of all programs needed to be loaded
     in start up (reminds you of anything?).
  
  5. A region of virtual address space (usually 4GB) is allocated for each
     start-up component (including the kernel) by modifying
     the UEFI page table.
     
  6. Each start-up component is loaded to the physical memory,
     and its logical address space is initialized in the great
     page table. It is the responsibility of the loader component
     to load the code, data, stack, heap and other sections
     to the correct addresses (relative to region's base address)
     as specified in the binary files.
     
  7. A table listing all the loaded components is created into memory, and
     passed to ServMan through ServMan's stack.

  8. Now ServMan is loaded to memory and has its call arguments set up
     and everything is cool... The next step is to jump from the bootloader
     to ServMan's entry point. ServMan doesn't need to load any program
     from any disk because it is already done by the bootloader (gratefully!).
     It just initializes its internal IPC component, and uses this component
     to jump between (the already loaded) programs and that's it.
     
  9. After ServMan is initialized, it starts to initalize other components.
     This includes MemMan (memory manager), DevMan (device manager),
     FileMan (file system), device drivers, file-system drivers, etc.
     For example, DevMan doesn't need to depend on any filesystem
     or any previous device driver, because all needed modules needed
     to start up the system are already loaded.
     
  10. All the calls between ServMan and other programs are simple IPC messages
      handled by the IPC component of ServMan itself. We don't need any magic.

