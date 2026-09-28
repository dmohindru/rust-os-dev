# x86-64 Exceptions, GDT, TSS, and IST — Discussion Summary

## 1. Exception handling and the stack

A regular function call uses the stack to manage execution. On x86-64, `call` normally pushes a return address, and functions may use stack space for local variables, saved registers, and other data. The compiler may instead keep variables in registers or optimize them away, so not every local variable necessarily lives on the stack.

Exception entry has a related but distinct purpose: the CPU must preserve enough interrupted processor state to enter an exception handler. For typical x86-64 exception entry, the CPU saves an exception frame on a stack; some exceptions also push an error code. The exact frame depends on the exception and whether a privilege-level or IST stack switch occurs.

Exception delivery is not simply an ordinary `call` to a Rust function. The CPU follows its architectural exception-entry rules, and handlers typically return with `iretq`, not `ret`.

## 2. Why a dedicated exception stack can be necessary

If the current stack is invalid, exhausted, or unmapped, the CPU may be unable to save the exception frame there. This can cause another exception while trying to deliver the first one.

A double-fault handler therefore needs a way to begin on a separate, known-good stack. Hardware-assisted stack switching lets the CPU switch stacks as part of exception delivery—before saving the exception frame on the selected stack. A software switch inside the handler would be too late if the CPU cannot complete entry in the first place.

The emergency stack itself must still be valid, mapped, writable, and sufficiently large. IST does not guarantee recovery if the emergency stack or handler also fails.

## 3. IDT, IST, TSS, and GDT: what they are

These are primarily architecture-defined data structures in memory, supported by special CPU registers and instructions.

| Name | Meaning | What it is |
|---|---|---|
| **IDT** | Interrupt Descriptor Table | A table in memory containing gates for exceptions and interrupts. A gate identifies a handler and can select an IST entry. |
| **GDT** | Global Descriptor Table | A table in memory containing segment and system descriptors, including a TSS descriptor. |
| **TSS** | Task State Segment | A memory structure. In 64-bit mode, a key use is supplying privilege-level stack pointers and IST pointers. |
| **IST** | Interrupt Stack Table | Seven stack-pointer entries *inside the 64-bit TSS*. It is not a separate CPU register or independent table. |
| **GDTR** | Global Descriptor Table Register | CPU register holding the GDT base address and limit. |
| **IDTR** | Interrupt Descriptor Table Register | CPU register holding the IDT base address and limit. |
| **TR** | Task Register | CPU register holding the active TSS selector, with associated cached descriptor state. |

The OS chooses where to allocate the GDT, IDT, TSS, and stack memory. They are not located at universal fixed addresses.

## 4. The hardware relationships

The core connection is:

**GDT → TSS descriptor → active TSS → IST stack pointer → stack memory**

And for exception delivery:

**IDT gate → selected IST index → TSS's corresponding IST pointer → CPU switches stack → handler begins**

The GDT makes the TSS descriptor available to the CPU. The task register identifies the active TSS. The IDT says which handler to invoke and, if configured, which IST slot to use.

## 5. What the instructions do

- **`lgdt`** reads a memory operand containing a GDT base and limit, then loads those values into **GDTR**. It does not copy the whole GDT into a CPU register.
- **`lidt`** similarly loads the IDT base and limit into **IDTR**.
- **`ltr`** loads a selector identifying the TSS descriptor into **TR** and establishes the CPU's active TSS state.

The tables remain in memory. The CPU consults their descriptors when required and maintains relevant internal/cached state. Segment registers also have visible selectors and associated hidden descriptor state. The CPU does not copy the entire GDT into a hidden register bank.

## 6. TSS: privilege stacks versus IST

The 64-bit TSS includes:

- **RSP0, RSP1, RSP2:** stack pointers associated with privilege levels. In the common user-to-kernel transition, RSP0 supplies the ring-0 stack.
- **IST1–IST7:** dedicated stack pointers that an IDT entry can request for a particular interrupt or exception.
- **I/O map base:** supports the optional I/O permission bitmap.

RSP0 and IST are related but not interchangeable:
- RSP0 supports a privilege-level stack transition, such as entering ring 0 from ring 3.
- IST allows a particular IDT entry to select a designated stack, including when no privilege-level change is involved.

## 7. Example: double fault with IST

A simplified path:

1. Kernel code is executing on its current stack.
2. A fault occurs—for example, a stack overflow touches an unmapped guard page.
3. The CPU attempts to deliver the page fault. If exception delivery itself faults under the architectural conditions for double fault, the CPU raises `#DF`.
4. The double-fault IDT gate specifies an IST slot, such as slot 0.
5. The CPU obtains that stack pointer from the active TSS and switches to the dedicated stack.
6. The CPU saves the double-fault exception frame on that stack and transfers control to the handler.

A page fault in kernel mode does **not** automatically cause a double fault. Many kernel page faults can be handled normally. A double fault occurs only when the relevant exception-delivery conditions are met.

If delivery of the double-fault handler also fails, a triple fault can result; emulators such as QEMU commonly respond by resetting the virtual machine.

## 8. Kernel virtual memory and page faults

Virtual addresses are used in both user mode and kernel mode when paging is enabled. Page tables define mappings and permissions for both.

Kernel privilege does not mean every virtual address is mapped or every access is allowed. Kernel stacks are also memory regions that need valid mappings.

A page fault can occur in kernel mode due to, for example:
- Accessing a non-present or unmapped page (such as a guard page).
- Writing to a read-only page.
- Violating other paging permissions or restrictions.

“Unmapped” means the active page tables do not provide a valid mapping for the attempted access; it does not mean the virtual address value itself does not exist.

The CPU records the faulting virtual address in **CR2** and supplies a page-fault error code with details about the access.

## 9. Guard-page intuition

A kernel can deliberately leave a page adjacent to a stack unmapped. If the stack grows into it, the resulting page fault can detect a stack overflow. This is a deliberate safety mechanism, not necessarily an accidental missing mapping.

The guard page can also explain why a fault handler may need an independent stack: the ordinary stack may no longer be safe for exception entry.

## 10. Philipp Oppermann tutorial code: conceptual mapping

The tutorial's setup follows this pattern:

1. Reserve actual memory for a dedicated double-fault stack.
2. Compute its top address (x86-64 stacks grow downward).
3. Store that address in a TSS IST entry.
4. Add a TSS descriptor to the GDT.
5. Load the GDT (`lgdt`) and the TSS selector (`ltr`).
6. Configure the double-fault IDT gate to use the chosen IST slot.

The static stack array is the memory; the TSS only stores its pointer. The IDT selects the slot; it does not contain the stack itself.

## Quick mental model

- **GDT:** registers the TSS descriptor.
- **TSS:** holds stack-pointer configuration, including IST.
- **IDT:** chooses the exception handler and optional IST slot.
- **GDTR / IDTR / TR:** CPU registers that establish access to those structures.
- **IST:** lets the CPU enter selected handlers on a separate stack.
- **Page fault:** can happen in kernel mode because kernel code also accesses virtual memory.

**One-line summary:** The GDT makes the TSS available, the TSS tells the CPU where the emergency stack is, and the IDT tells the CPU when to use it.
