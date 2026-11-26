# Writing an OS in Rust --- Chapter-wise Summary and Raspberry Pi 4 Study Guide

A consolidation guide for **Philipp Oppermann's *Writing an OS in
Rust*** and a preparation roadmap for **Rust Embedded's Raspberry Pi OS
tutorials**.

-   Original Rust OS tutorial: https://os.phil-opp.com/
-   Raspberry Pi OS tutorials:
    https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials

------------------------------------------------------------------------

## 1. How the two tutorial series fit together

Philipp Oppermann's series teaches foundational operating-system
mechanisms, primarily using the x86-64 architecture. The Raspberry Pi
series applies many of the same OS concepts to a real ARMv8-A / AArch64
platform.

The conceptual knowledge transfers, but architecture-specific
implementations differ:

-   x86-64 uses its own boot conventions, exception mechanisms,
    interrupt controllers and paging structures.
-   Raspberry Pi 4 uses ARMv8-A, AArch64, exception levels, exception
    vectors, a Generic Interrupt Controller (GIC), and ARM translation
    tables.
-   Rust concepts such as `no_std`, ownership, `Future`, allocators and
    safe abstractions remain relevant across architectures.

For each chapter, focus on: 1. What problem does it solve? 2. Which
mechanisms and concepts are introduced? 3. What should you be able to
explain or implement afterwards? 4. How does the knowledge transfer to
Raspberry Pi OS development?

# Part I --- Writing an OS in Rust

The series can be viewed as four broad stages:

1.  **Bare Bones:** build and run a freestanding kernel.
2.  **Interrupts:** handle CPU exceptions and external hardware events.
3.  **Memory Management:** understand virtual memory, paging and
    allocation.
4.  **Multitasking:** use Futures, Wakers and an executor for
    cooperative task execution.

## Section A --- Bare Bones

### Chapter 1: A Freestanding Rust Binary

Source: https://os.phil-opp.com/freestanding-rust-binary/

**Problem:** How can Rust code execute without an underlying operating
system?

Ordinary Rust applications use the standard library (`std`), which
depends on OS services. A bare-metal kernel cannot make those
assumptions.

**Important concepts** - `#![no_std]`: avoid depending on the Rust
standard library. - `#![no_main]`: take control of the program entry
point. - `core`: Rust functionality available without an OS. - Panic
handling: provide a panic handler. - Target configuration, linker
configuration and executable layout. - Entry point and calling
conventions.

A freestanding kernel needs an entry point and runtime environment
appropriate to its boot process. A conventional `main()` function is not
automatically the kernel entry point.

**What to retain** - Compiling code and getting a processor to execute
it are different problems. - The compiler and linker produce an
executable, but startup code or a bootloader must establish the initial
execution environment. - `no_std` does not mean Rust has no useful
library functionality; `core` remains available.

**Raspberry Pi relevance:** Very high. Cross-compilation, linker
scripts, entry points and AArch64 startup code become central.

### Chapter 2: A Minimal Rust Kernel

Source: https://os.phil-opp.com/minimal-rust-kernel/

**Problem:** How do we turn a freestanding executable into a bootable
kernel?

**Important concepts** - Boot process and bootloader responsibilities. -
Kernel entry point. - Bootable images. - Linker and executable layout. -
Initial CPU and memory state. - Initial paging and address-space
assumptions.

A bootloader prepares the machine and transfers control to the kernel.
The kernel must know what state the bootloader guarantees.

**What to retain** - Understand the sequence from machine startup,
through firmware or bootloader, to kernel entry. - Know which setup is
performed before Rust code begins. - Be aware that the kernel's
assumptions about memory and processor state must match the boot
environment.

**Raspberry Pi relevance:** Very high. Raspberry Pi firmware and its
boot process differ from the x86 boot arrangements in this chapter, but
the responsibilities are analogous.

### Chapter 3: VGA Text Mode

Source: https://os.phil-opp.com/vga-text-mode/

**Problem:** How can a kernel produce output without an OS-provided
display API?

The chapter accesses VGA text memory directly.

**Important concepts** - Memory-mapped I/O (MMIO). - Volatile memory
access. - Raw pointers and `unsafe`. - Encapsulating unsafe code behind
safe interfaces. - Formatting using `core::fmt::Write`. -
Synchronisation around shared output.

**What to retain** - Hardware registers and device memory need
appropriate access semantics. - `volatile` prevents the compiler from
treating hardware accesses like ordinary memory accesses that can be
freely removed or optimised. - Unsafe operations should be contained
behind small, carefully designed abstractions. - VGA text mode is a
particular x86 facility, not a general display mechanism.

**Raspberry Pi relevance:** High for MMIO and safe hardware
abstractions; the VGA mechanism itself does not transfer. Raspberry Pi
output commonly begins with UART.

### Chapter 4: Testing

Source: https://os.phil-opp.com/testing/

**Problem:** How can kernel code be tested without the ordinary OS
environment expected by Rust's standard test harness?

**Important concepts** - Unit and integration testing. - Custom test
frameworks and test runners. - `#[test_case]`. - QEMU as a test
environment. - Kernel-specific panic and test-result reporting.

**What to retain** - Testing should be part of kernel development from
the beginning. - Generic algorithms can often be tested independently of
physical hardware. - Hardware-dependent functionality benefits from a
testable abstraction boundary. - QEMU can provide a repeatable
environment for many kernel tests, although it cannot replace all
physical-hardware testing.

**Raspberry Pi relevance:** High. Apply the same testing discipline
while adapting to the Raspberry Pi project's build and test setup.

## Section B --- Interrupts

Interrupts change the execution model: the processor can transfer
control to a handler in response to exceptions or external events.

### Chapter 5: CPU Exceptions

Source: https://os.phil-opp.com/cpu-exceptions/

**Problem:** How does the CPU notify the kernel that an exceptional
condition has occurred?

Examples include division by zero, invalid memory access, breakpoints
and invalid instructions.

**Important concepts** - Interrupt Descriptor Table (IDT). - Exception
handlers. - Exception frames. - Exception return. - Breakpoint
exceptions. - Synchronous exceptions versus asynchronous hardware
interrupts.

**What to retain** - The processor provides a mechanism for transferring
control to a handler. - The kernel defines what to do when an exception
occurs. - Some exceptions are recoverable; others indicate a fatal
kernel problem.

**Raspberry Pi relevance:** Very high conceptually. ARM uses exception
vectors and exception syndrome registers rather than the x86 IDT.

### Chapter 6: Double Faults

Source: https://os.phil-opp.com/double-fault-exceptions/

**Problem:** What happens if an exception occurs while the processor is
trying to handle another exception?

**Important concepts** - Nested exceptions. - Double faults and
triple-fault behaviour. - Stack overflow and corrupted-stack
scenarios. - Interrupt Stack Table (IST). - Dedicated exception stacks.

**What to retain** - Exception handling can itself fail. - Critical
handlers need a reliable execution environment. - Dedicated stacks can
help handle faults that occur when the normal stack is unusable.

**Raspberry Pi relevance:** High. ARM has a different exception-entry
model, but exception stacks and robust fault handling remain important.

### Chapter 7: Hardware Interrupts

Source: https://os.phil-opp.com/hardware-interrupts/

**Problem:** How can the kernel respond to external hardware events
without continuously polling every device?

**Important concepts** - Programmable Interrupt Controller (PIC). -
Hardware IRQs. - Interrupt handlers. - Timer and keyboard interrupts. -
Interrupt acknowledgement / EOI. - Interrupt masking and enabling. -
Difference between CPU exceptions and hardware interrupts.

**What to retain** A typical interrupt flow is:

1.  Hardware raises an interrupt request.
2.  The interrupt controller identifies and routes it.
3.  The CPU transfers control to the handler.
4.  The handler processes or acknowledges the event.
5.  The CPU returns to interrupted execution when appropriate.

**Raspberry Pi relevance:** Extremely high. Raspberry Pi 4 uses the ARM
Generic Interrupt Controller (GIC), not the x86 PIC.

## Section C --- Memory Management

### Chapter 8: Introduction to Paging

Source: https://os.phil-opp.com/paging-introduction/

**Problem:** How can an OS manage memory, translate addresses and
isolate different parts of a system?

**Important concepts** - Physical and virtual memory. - Memory
isolation. - Segmentation and paging. - Page frames and page tables. -
Multilevel page tables. - Address translation. - Memory fragmentation.

The CPU generates virtual addresses. The MMU uses page tables to
translate them to physical addresses.

**What to retain** - Why virtual memory exists. - How virtual pages map
to physical frames. - Why multilevel page tables are used. - How page
permissions contribute to memory protection.

**Raspberry Pi relevance:** Extremely high. The concepts transfer, but
ARM's translation tables and configuration registers differ from x86-64.

### Chapter 9: Paging Implementation

Source: https://os.phil-opp.com/paging-implementation/

**Problem:** How do we implement page-table manipulation and address
translation?

**Important concepts** - Physical frame access and allocation. -
Page-table structures and walking. - Mapping and unmapping virtual
addresses. - Page-table entries and flags. - Identity mapping. -
Accessing page tables themselves.

A key implementation challenge is that page tables occupy physical
memory, while the kernel normally accesses memory through virtual
addresses. The kernel needs a strategy for accessing and manipulating
its page tables.

**What to retain** - Understand how a virtual address is translated. -
Understand how page-table entries encode frame addresses and
permissions. - Understand mapping, unmapping and the need to maintain
translation state correctly.

**Raspberry Pi relevance:** Extremely high. You will implement AArch64
translation tables and configure the ARM MMU.

### Chapter 10: Heap Allocation

Source: https://os.phil-opp.com/heap-allocation/

**Problem:** How can a kernel support dynamic allocation?

Rust's `alloc` types, including `Box`, `Vec` and `String`, require an
allocator.

**Important concepts** - Stack versus heap. - Static versus dynamic
allocation. - `GlobalAlloc`. - Allocation and deallocation. - Heap
memory region. - `alloc` crate. - Alignment.

**What to retain** There are separate responsibilities: 1. Establish a
memory region that is mapped and reserved for the heap. 2. Implement an
allocator that manages that region. 3. Expose the allocator through
Rust's allocation interface.

**Raspberry Pi relevance:** High. The allocator interface is similar,
but the memory layout and MMU setup are architecture-specific.

### Chapter 11: Allocator Designs

Source: https://os.phil-opp.com/allocator-designs/

**Problem:** How should a heap allocator organise and manage free
memory?

The chapter explores three designs:

**1. Bump allocator** - Advances a pointer through a memory region. -
Very fast and simple. - Individual deallocation is generally
unsupported. - Useful when memory can be reclaimed as a whole.

**2. Linked-list allocator** - Represents free memory regions as linked
nodes. - Searches for a sufficiently large region. - Supports
deallocation by returning regions to the free list. - Must deal with
fragmentation and potentially coalescing adjacent free regions.

**3. Fixed-size block allocator** - Divides memory into predefined block
sizes. - Can make common allocations fast. - May waste space through
internal fragmentation.

**What to retain** Compare allocation speed, deallocation complexity,
internal and external fragmentation, metadata overhead, alignment and
predictability.

**Raspberry Pi relevance:** High. The same allocator principles apply on
ARM.

## Section D --- Multitasking

### Chapter 12: Async/Await

Source: https://os.phil-opp.com/async-await/

**Problem:** How can a kernel cooperatively execute multiple tasks
without blocking while they wait for events?

**Important concepts** - Cooperative multitasking. - `Future` and
`Poll`. - Compiler-generated async state machines. - `Pin`. -
`Context`. - `Waker` and `Wake`. - Task scheduling. - Executor. -
Interrupt-driven notifications.

**The three central responsibilities**

  -----------------------------------------------------------------------
  Entity                              Responsibility
  ----------------------------------- -----------------------------------
  Future                              Stores its progress and determines
                                      what to do when polled

  Waker                               Notifies the runtime that a task
                                      may be able to make progress

  Executor                            Schedules tasks and calls `poll()`
                                      on ready tasks
  -----------------------------------------------------------------------

**Important execution model** 1. The executor polls a Future. 2. The
Future progresses until it completes or reaches an operation that
returns `Pending`. 3. The asynchronous operation registers the current
Waker when it needs to wait. 4. An event source, such as an interrupt
handler, makes progress possible and invokes the Waker. 5. The Waker
causes the task to be scheduled again. 6. The executor polls the Future
again. 7. The Future's stored state determines where execution
continues.

A Waker does not directly resume a Future or execute its code. It
notifies the runtime so the executor can poll the task again.

**What to retain** - How an async function becomes a state machine. -
Why local variables may need to live inside the Future. - What `Pending`
and `Ready` mean. - How a Waker connects an event source to task
scheduling. - Why an executor does not need to know the internal state
of each Future.

**Raspberry Pi relevance:** High. Interrupts and timers provide the
event sources needed for an embedded async runtime. A production runtime
also needs careful task ownership, synchronisation, wake-race handling,
queue management and idle behaviour.

------------------------------------------------------------------------

# Part II --- What the first series has taught

The 12 chapters are not unrelated topics. Together, they establish the
foundations of a kernel.

  -----------------------------------------------------------------------
  Area                                Fundamental question
  ----------------------------------- -----------------------------------
  Boot                                How does the CPU start executing my
                                      kernel?

  Runtime                             What environment must be
                                      established before Rust runs?

  Hardware                            How does software communicate with
                                      physical devices?

  Exceptions                          How does the CPU report exceptional
                                      conditions?

  Interrupts                          How does hardware notify the CPU of
                                      events?

  Memory                              How does the kernel manage physical
                                      and virtual memory?

  Allocation                          How does the kernel manage
                                      dynamically allocated memory?

  Multitasking                        How does the kernel manage multiple
                                      independent tasks?
  -----------------------------------------------------------------------

Three especially important foundations to carry forward:

1.  **Hardware/software boundaries:** the CPU, peripherals and kernel
    are distinct entities that communicate through defined mechanisms.
2.  **Memory management:** memory is more than an array of bytes;
    physical memory, virtual addresses, page tables, alignment and
    allocation all matter.
3.  **Event-driven execution:** hardware produces events, interrupt
    handlers respond, and the kernel decides what work should happen
    next.

# Part III --- Raspberry Pi 4 OS tutorials

Repository:
https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials

The Raspberry Pi project is authored by Andre Richter. It teaches OS
development on Raspberry Pi hardware using Rust and AArch64.

The project progresses incrementally and separates generic kernel code,
architecture-specific code and board-support functionality.

## The broad architecture

A useful way to think about the separation is:

-   **Generic kernel:** architecture-independent kernel functionality.
-   **Architecture-specific implementation:** AArch64 CPU operations,
    MMU and exception handling.
-   **Board Support Package (BSP):** Raspberry Pi hardware and
    peripheral details.

This is a useful architectural pattern for a kernel of your own.

## Tutorial roadmap

The following is a concise guide to the repository's tutorial
directories and their main subjects. Consult the repository README and
source for the exact implementation details.

  Tutorial   Subject                 Main learning objective
  ---------- ----------------------- --------------------------------
  00         Before we start         Environment and architecture
  01         Wait forever            First bare-metal kernel
  02         Runtime init            Initial runtime setup
  03         Hacky hello world       First output
  04         Safe globals            Safe global state
  05         GPIO and UART drivers   Hardware interaction
  06         UART chainloader        Loading kernels over UART
  07         Timestamps              Time measurement
  08         Hardware debugging      JTAG and GDB
  09         Privilege level         ARM exception levels
  10         Virtual memory I        Identity mapping
  11         Exceptions I            Exception handling groundwork
  12         Integrated testing      Kernel testing
  13         Exceptions II           Peripheral interrupts
  14         Virtual memory II       MMIO remapping
  15         Virtual memory III      Precomputed translation tables
  16         Virtual memory IV       Higher-half kernel
  17         Kernel symbols          Symbol information
  18         Backtrace               Stack backtracing
  19         Kernel heap             Dynamic memory
  20         Timer callbacks         Timer-based callbacks

### Stage 0 --- Environment and bootstrapping

#### Tutorial 00: Before We Start

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/00_before_we_start

Study: - ARMv8-A and AArch64. - Cross-compilation. - QEMU. - Build
tooling and project structure. - Raspberry Pi hardware requirements.

Understand the project architecture before diving into the
implementation.

#### Tutorial 01: Wait Forever

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/01_wait_forever

The first kernel boots and remains running.

Study: - Kernel entry point. - Assembly startup code. - Initial CPU
state. - Stack initialisation. - Infinite execution loop. - What the CPU
does when the kernel has nothing to do.

#### Tutorial 02: Runtime Init

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/02_runtime_init

Study: - CPU state. - Stack setup. - BSS initialisation. - Static
data. - Runtime entry and initialisation order.

Rust assumes certain runtime conditions. Startup code must establish
them before ordinary Rust code runs.

#### Tutorial 03: Hacky Hello World

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/03_hacky_hello_world

The kernel begins producing visible output.

Focus on how the kernel communicates with hardware without an
OS-provided output API. Treat this as a stepping stone to the UART
driver.

#### Tutorial 04: Safe Globals

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/04_safe_globals

Study: - Global variables and static storage. - Interior mutability. -
Synchronisation. - Safe abstractions around mutable global state. - Rust
ownership and concurrency guarantees.

This connects to embedded Rust patterns such as `static_cell`, mutexes
and shared peripheral buses.

### Stage 1 --- Hardware interaction

#### Tutorial 05: Drivers --- GPIO and UART

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/05_drivers_gpio_uart

A major hardware milestone.

Study: - GPIO and UART registers. - MMIO. - Peripheral drivers. -
Register-level interaction. - BSP abstractions.

Pay attention to how the project structures driver interfaces. Compare
the separation of hardware-specific operations and higher-level APIs
with the design of your own embedded drivers.

#### Tutorial 06: UART Chainloader

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/06_uart_chainloader

Study: - UART binary transfer. - Kernel image loading. - Memory
placement. - Bootloader and chainloader responsibilities. - Control
transfer.

This provides a useful introduction to loading executable images outside
a conventional OS environment.

#### Tutorial 07: Timestamps

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/07_timestamps

Study: - Hardware counters. - Timer registers. - Clock frequency. -
Timestamp representation. - Duration conversion.

Distinguish a hardware counter, a clock frequency, a duration and
wall-clock time. These concepts matter for timeouts and scheduling.

#### Tutorial 08: Hardware Debugging --- JTAG

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/08_hw_debug_JTAG

Study: - JTAG. - OpenOCD. - GDB. - Breakpoints. - CPU registers. -
Memory inspection. - Stepping through assembly.

Hardware debugging is especially useful for page-table errors, exception
handling, stack corruption and unexpected CPU state.

### Stage 2 --- CPU architecture and exceptions

#### Tutorial 09: Privilege Level

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/09_privilege_level

ARMv8-A has four exception levels:

  Level   Typical role
  ------- ---------------------------
  EL0     User applications
  EL1     Operating-system kernel
  EL2     Hypervisor
  EL3     Secure monitor / firmware

Study: - Exception levels. - Transitions between levels. -
System-register access. - Why kernels commonly operate at EL1. -
Privilege separation.

The ARM model is conceptually related to x86 privilege rings, but it is
architecturally different. Startup code must account for the exception
level at which the boot environment enters the kernel.

### Stage 3 --- Virtual memory and interrupts

#### Tutorial 10: Virtual Memory Part 1 --- Identity Mapping

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/10_virtual_mem_part1_identity_mapping

Study: - MMU. - Translation tables. - Identity mapping. - Virtual and
physical addresses. - Translation granules. - Page-table descriptors.

Identity mapping means the virtual address equals the physical address
(`VA = PA`). It can simplify the transition when enabling the MMU.

#### Tutorial 11: Exceptions Part 1 --- Groundwork

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/11_exceptions_part1_groundwork

Study: - Exception vectors. - Exception vector table. - Synchronous
exceptions. - IRQ and FIQ. - Exception syndrome registers. - Saved
processor state. - Exception return.

ARM's exception vector table is different from the x86 IDT.

#### Tutorial 12: Integrated Testing

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/12_integrated_testing

Study: - Testing kernel components. - Integrated test execution. -
Separating generic and hardware-specific functionality. - Verifying
behaviour as the kernel grows.

#### Tutorial 13: Exceptions Part 2 --- Peripheral IRQs

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/13_exceptions_part2_peripheral_IRQs

Study: - Generic Interrupt Controller (GIC). - Peripheral interrupt
sources. - Interrupt routing. - IRQ handlers. - Interrupt
acknowledgement. - Interrupt masking and enabling.

This is a particularly important milestone for anyone interested in an
asynchronous runtime. The interrupt controller and handlers provide the
mechanism for receiving hardware events.

#### Tutorial 14: Virtual Memory Part 2 --- MMIO Remap

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/14_virtual_mem_part2_mmio_remap

Study: - MMIO address mapping. - Device memory attributes. -
Cacheability. - Memory ordering. - Mapping peripheral regions.

Normal RAM and device registers should not necessarily use the same
memory attributes. Incorrect caching or memory attributes can cause
hardware behaviour to break.

#### Tutorial 15: Virtual Memory Part 3 --- Precomputed Tables

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/15_virtual_mem_part3_precomputed_tables

Study: - Static translation tables. - Memory layout planning. -
Pre-initialised structures. - Translation-table alignment. - Page-table
descriptors.

This demonstrates how an initial memory layout can be established
without requiring a fully dynamic page-table manager.

#### Tutorial 16: Virtual Memory Part 4 --- Higher-Half Kernel

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/16_virtual_mem_part4_higher_half_kernel

Study: - Higher-half kernel layout. - Kernel virtual address space. -
Separation of kernel and lower address spaces. - Consistent virtual
memory layout. - Memory isolation.

A higher-half kernel is mapped into a deliberately separated region of
virtual address space.

### Stage 4 --- Diagnostics and runtime services

#### Tutorial 17: Kernel Symbols

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/17_kernel_symbols

Study: - ELF symbol tables. - Debug information. - Linker symbols. -
Symbol resolution. - Kernel diagnostics.

Symbols help translate machine addresses into meaningful function and
variable names.

#### Tutorial 18: Backtrace

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/18_backtrace

Study: - Stack frames. - Frame pointers. - Call chains. - Stack
unwinding. - Symbol resolution.

A backtrace helps reconstruct the call sequence that led to a failure or
execution point.

#### Tutorial 19: Kernel Heap

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/19_kernel_heap

Study: - Heap memory region. - Allocator implementation. - Allocation
interfaces. - Memory layout. - Dynamic memory management.

Compare this with Philipp's heap and allocator chapters. Understand how
the kernel reserves and maps memory before exposing it to Rust's
allocator interface.

#### Tutorial 20: Timer Callbacks

https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials/tree/master/20_timer_callbacks

Study: - Hardware timers. - Timer interrupts. - Deadline management. -
Callback registration. - Time-based event scheduling. - Deferred
execution.

A timer callback system and a Future executor are related but not
identical. A callback system schedules functions associated with timer
events; an async executor polls Futures and uses Wakers to reschedule
tasks when their underlying operations may progress.

------------------------------------------------------------------------

# Part IV --- Mapping the two series

  -----------------------------------------------------------------------
  Philipp's tutorial      Raspberry Pi tutorial   Main architectural
                                                  difference
  ----------------------- ----------------------- -----------------------
  Freestanding Rust       01--02                  Startup and runtime
  binary                                          differ

  Minimal Rust kernel     01--03                  Boot mechanism differs

  VGA text mode           03, 05                  UART instead of VGA

  Testing                 04, 12                  Different test
                                                  environment

  CPU exceptions          11                      ARM exception vectors

  Double faults           11                      ARM fault-handling
                                                  model

  Hardware interrupts     13                      GIC instead of PIC

  Introduction to paging  10                      ARM translation tables

  Paging implementation   10, 14--16              AArch64 MMU
                                                  configuration

  Heap allocation         19                      Similar Rust allocator
                                                  interface

  Allocator designs       19                      Similar allocator
                                                  algorithms

  Async/Await             13, 20                  Hardware events and
                                                  timers provide
                                                  foundations
  -----------------------------------------------------------------------

The Raspberry Pi tutorials should not be treated as a repetition of the
first series. The first series taught the OS concepts; the second gives
you the opportunity to understand how those concepts are implemented on
ARM.

# Part V --- Recommended study priorities

## Study deeply

-   01--02: Boot and runtime initialisation.
-   04: Safe global state.
-   05: GPIO and UART drivers.
-   09: ARM privilege levels.
-   10: AArch64 virtual memory.
-   11: ARM exception handling.
-   13: Interrupt controller and peripheral IRQs.
-   14--16: Advanced virtual memory.

## Understand the implementation

-   03: Basic output.
-   06: UART chainloader.
-   07: Timestamps.
-   08: JTAG debugging.
-   12: Integrated testing.
-   19: Kernel heap.
-   20: Timer callbacks.

## Supporting knowledge

-   17: Kernel symbols.
-   18: Backtrace.

## Five ARM topics to study alongside the tutorials

  -----------------------------------------------------------------------
  Topic                               Why it matters
  ----------------------------------- -----------------------------------
  AArch64 assembly                    Startup code, registers and
                                      function calls

  ARM exception model                 Exception levels and exception
                                      transitions

  GIC                                 Hardware interrupt routing

  ARM MMU                             Translation tables and memory
                                      attributes

  Raspberry Pi peripheral             CPU and peripheral communication
  architecture                        
  -----------------------------------------------------------------------

Useful references:

-   ARM Cortex-A72 MPCore Technical Reference Manual:
    https://developer.arm.com/documentation/ddi0488/latest/
-   ARM Architecture Reference Manual (ARMv8-A):
    https://developer.arm.com/documentation/ddi0487/latest/
-   Raspberry Pi documentation:
    https://www.raspberrypi.com/documentation/

Use these manuals as references when you encounter a particular
register, instruction or hardware mechanism. You do not need to read
them cover to cover.

# Part VI --- A possible architecture for your own kernel

As you progress, consider gradually developing your own kernel
architecture rather than only reproducing the tutorial implementation.

A useful conceptual separation is:

``` text
Kernel application layer
    |
    v
Core kernel
    - Task management
    - Memory management
    - Scheduling
    |
    +-----------------------+
    |                       |
    v                       v
Architecture layer      Driver layer
    - AArch64 CPU           - UART
    - MMU                   - GPIO
    - Exceptions            - Timers
    - GIC                    - SPI
                             - Storage
    |                       |
    +-----------+-----------+
                |
                v
          Raspberry Pi 4
```

This is a proposed learning architecture, not a description of the
repository's exact module hierarchy.

Potential future experiments: - Your own interrupt-driven asynchronous
executor. - A simple timer service. - A no-heap task scheduler. - A
physical memory allocator. - A page-frame allocator. - A virtual memory
manager. - A simple filesystem backed by external SPI storage.

These are natural extensions after the foundational tutorials. They do
not all need to be implemented while following the series.

# Final revision checklist

Use this to test whether you have retained the important ideas from the
first series.

-   [ ] I can explain how a freestanding Rust binary differs from an
    ordinary application.
-   [ ] I can explain the bootloader's responsibilities and the kernel
    entry point.
-   [ ] I understand MMIO, volatile access and why unsafe hardware
    operations need safe abstractions.
-   [ ] I can explain the difference between CPU exceptions and hardware
    interrupts.
-   [ ] I understand why exception handlers need reliable stacks and
    execution environments.
-   [ ] I can explain virtual versus physical addresses.
-   [ ] I understand page tables and address translation.
-   [ ] I can explain the relationship between a heap region and an
    allocator.
-   [ ] I understand the trade-offs of bump, linked-list and fixed-size
    block allocators.
-   [ ] I can explain how an async function becomes a Future state
    machine.
-   [ ] I understand how Wakers notify an executor and how the executor
    polls a task again.

## The central mental model

An operating system is built by establishing a sequence of mechanisms:

1.  **Boot:** get the processor into a known execution environment.
2.  **Runtime:** initialise the state required by the kernel.
3.  **Hardware:** communicate with devices through well-defined
    interfaces.
4.  **Exceptions:** handle exceptional processor conditions.
5.  **Interrupts:** respond to external events.
6.  **Memory:** manage physical and virtual address spaces.
7.  **Allocation:** manage dynamic memory where needed.
8.  **Multitasking:** schedule and coordinate independent tasks.

The goal of the Raspberry Pi series is not simply to repeat these ideas,
but to understand how they are realised on ARMv8-A and how to use them
as building blocks for an OS of your own.
