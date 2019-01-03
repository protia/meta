
# Build Sequence

## I. Pre-build: Build Systema (on UNIX only)

You need to build Systema and install it on UNIX before
you go. This step is not needed if you are compiling
Protia from inside Protia itself, as Protia already
contains a Systema compiler.

## II. Build the System

### 1. Introduction to the package system

The system consists of a set of package groups. Each package group contains
a set of correlated packages. Examples for package groups:

 * Base: contains Base/Structs, Base/ServMan, Base/MemMan, Base/FileMan, etc.
 * Systema: contains Systema/Frontend, Systema/Assembler, Systema/Loader, etc.
 * Boot: contains Boot/Loader

### 2. Dependency system

Each package may depend on a set of other packages, but the resulting 
dependency graph shouldn't be non-circular (as we  describe below). For 
example, if the boot loader uses Systema/Loader to load start-up 
programs into memory, it needs to specify in its source code that it 
depends on systema's loader.

Strictly speaking, all the components in the system are ordinary programs,
libraries, etc. All programs assume the same memory model before they start 
execution: a linear address space containing code, data, stack, heap,
and other segments. Refer to the "bootloader" subsection below to see
how this requirement is satisfied for early start-up programs.

Each package needs to describe:

  1. Pre-requisities for building.
  2. Pre-requisities for installation.

For example, the Base/Structs package needs Systema/Frontend for 
compiling, but doesn't need any pre-requisite for run-time.
It provides very basic data structures used by all other things.

### 3. Compile-time dependency resolution

If a package is compiled from UNIX, then the "compile-time pre-requisities"
needs to be installed on the host (UNIX), but the "run-time pre-requisities"
needs to be installed on the target Protia system which the package is
built for.

Therefore, in order to start the compilation process, Systema needs
to be installed, because all the packages in the system (including the
Server Manager package) claim Systema as a compile-time pre-requirement.

If Protia is compiled inside Protia itself, then we guarantee that the
system already includes a systema compiler, because Protia is designed
from scratch to be a self-hosting, self-contained modular system.

It is good to note that Systema is not just a compiler, but it is a full
building system for Protia, providing a macro processor, front-end compiler,
back-end compiler, assembler, linker, loader, and a packager program.

Systema provides a mechanism to allow the package to describe its compilation
steps and installation steps (just like Autotools, Meson, Gradle, etc.)
Therefore, for compilation in general, we only need Systema. We don't
need any other package to be built on UNIX.

### 4. Run-time dependency resolution

To explain how the run-time dependencies are resolved, let's say you are on 
UNIX and you have installed systema, now you can create the following 
directories:
 
  * src/    : where all the packages of Protia are
  * build/  : a folder for temporary build files
  * disk/   : this is the target Protia system that you are building.

If you want to install Protia on /dev/sda1 for example, you
can format /dev/sda1 with one of the file-systems supported
by Protia, then start the build and let Systema know that
disk/ is your target installation folder (like $PREFIX in autotools).
and that build/ is your temporary build folder.

It is important to note that, for each package, you need to let
Systema know that build/ is your build directory, and disk/
is your target directory.

FYI, Systema by default looks for run-time pre-requisities in 
"C:\Protia", but since we have changed our $PREFIX to disk/, the
compiler now will look for run-time pre-requisities from there.

Since Base/Structs requires only Systema for compile-time (which,
at this step, is already compiled on UNIX or already available on 
Protia), then this condition is satisfied. The run-time pre-requisites 
are satisfied by default because Base/Structs doesn't depend on any 
other library. Now you compile Base/Structs to build/, then
install it to disk/.

Next, you need to compile the next package in the dependency graph, which 
is Base/ServMan. Systema will look first for compile-time requirements, 
which are already satisified (Systema is already installed on
the host UNIX system). Next, Systema will look into disk/ to see
whether run-time requirements are installed or not. The run-time
requirements for Base/ServMan is solely Base/Structs. And because
it is already installed to disk/ in the previous step.

That way, we can build up the system package by package, starting
with ServMan, until we build all the packages. In the end,
disk/ will contain a copy of how the real system partition
looks like in the end.

### 5. Build Recipes

The packager program provides the same functionality provided by Makefile
on UNIX: it processes "RECIPE.???" files to compile an individual package,
and has the ability to install the compiled files to a target location.
The "RECIPE.???" file specifies all the information needed to achieve this.

That being set, Systema is self-contained and doesn't need another package
manager to be built in the host system (whether UNIX or Protia). So, all
what you need is to compile Systema if it is installed (i.e., if you
are on UNIX), then build the packages, starting with Base/Structs.

Each package in Protia provides a "RECIPE.???" file that specifies:

  1. Pre-requisities for compile-time.
  2. Pre-requisities for run-time.
  3. Executables, libraries, etc. that the package provides.
  4. How to compile them.
  5. How to install them.

### 6. Package Order

The following list provides package ordering and the (run-time) dependency graph:

```
 * Base/Structs
 * Base/ServMan      --> Base/Structs
 * Base/MemMan       --> Base/Structs, Base/ServMan
 * Base/FileMan      --> Base/Structs, Base/ServMan, Base/MemMan
 * Base/DevMan       --> Base/Structs, Base/ServMan, Base/MemMan
 * Drivers/PC        --> Base/Structs, Base/ServMan, Base/MemMan, Base/DevMan
 * Drivers/CPU       --> Base/Structs, Base/ServMan, Base/MemMan, Base/DevMan, Drivers/PC
 * Drivers/PCI       --> Base/Structs, Base/ServMan, Base/MemMan, Base/DevMan, Drivers/PC
 * Drivers/ATA       --> Base/Structs, Base/ServMan, Base/MemMan, Base/DevMan, Drivers/PCI, Base/FileMan
 * Drivers/...       --> Base/Structs, Base/ServMan, Base/MemMan, Base/DevMan, ...
 * Filesys/FAT       --> Base/Structs, Base/ServMan, Base/MemMan, Base/FileMan
 * Filesys/EXT2      --> Base/Structs, Base/ServMan, Base/MemMan, Base/FileMan
 * Filesys/...       --> Base/Structs, Base/ServMan, Base/MemMan, Base/FileMan, ...
 * Systema/Macro     --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Front     --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Back      --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Assembler --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Linker    --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Loader    --> Base/ServMan, Base/MemMan, Base/FileMan
 * Systema/Builder   --> Base/ServMan, Base/MemMan, Base/FileMan
 * Boot/Loader       --> Base/ServMan, Systema/Linker
```

## III. Automating the build

We provide a builder repo to automate the build process of Protia.
build.sh is provided (to be run from UNIX) and RECIPE.??? is provided
to automatically build Protia from inside Protia.
