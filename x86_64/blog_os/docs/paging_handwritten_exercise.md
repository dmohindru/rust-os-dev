# Paging Handwritten Exercise

A small paper-and-pencil exercise for understanding what the OS/Rust paging code is doing.

The exercise deliberately uses a simplified 3-level page table instead of x86-64's 4-level page table, while preserving the important ideas.

---

## Goal

Configure and walk page tables so that:

```text
Virtual address:  0x2B7
Physical address: 0x5B7
```

Do not look for the answer in this file. Work it out by hand.

---

## 1. Toy CPU

Our CPU uses a 12-bit virtual address:

```text
┌────────┬────────┬────────┬────────────┐
│   L3   │   L2   │   L1   │   Offset   │
│ 2 bits │ 2 bits │ 2 bits │   6 bits   │
└────────┴────────┴────────┴────────────┘
   11-10     9-8      7-6       5-0
```

Therefore:

- Each page is `2^6 = 64` bytes.
- Each page-table level has `2^2 = 4` entries.
- The bottom 6 bits are the page offset.
- The remaining bits select entries at L3, L2, and L1.

---

## 2. Target

We want this mapping:

```text
Virtual address:   0x2B7
Physical address:  0x5B7
```

First determine:

```text
L3 index = ?
L2 index = ?
L1 index = ?
offset   = ?
```

---

## 3. Physical memory

Physical memory is divided into 64-byte frames.

Frame addresses therefore begin at:

```text
0x000
0x040
0x080
0x0C0
0x100
0x140
0x180
0x1C0
0x200
...
0x580
0x5C0
...
```

The target physical address is:

```text
0x5B7
```

Remember that `0x5B7` is an address *inside* a frame, not the beginning of one.

The physical address therefore consists conceptually of:

```text
physical frame + offset
```

---

## 4. Root page table

The CPU's equivalent of `CR3` contains:

```text
CR3 = 0x100
```

Therefore:

```text
CR3
 │
 ▼
physical 0x100
 │
 ▼
L3 page table
```

---

## 5. L3 page table

The L3 table contains four entries:

| L3 index | Entry |
|---:|---|
| 0 | not present |
| 1 | physical `0x180` |
| 2 | physical `0x240` |
| 3 | not present |

An entry containing a physical address means:

> The next-level page table is located at this physical address.

`not present` means the mapping does not exist.

---

## 6. L2 tables

### L2 table at physical `0x180`

| L2 index | Entry |
|---:|---|
| 0 | physical `0x300` |
| 1 | not present |
| 2 | physical `0x380` |
| 3 | not present |

### L2 table at physical `0x240`

| L2 index | Entry |
|---:|---|
| 0 | not present |
| 1 | physical `0x400` |
| 2 | not present |
| 3 | physical `0x440` |

These addresses point to the next page-table level, not directly to final physical memory.

---

## 7. L1 tables

### L1 table at `0x300`

| L1 index | Physical frame |
|---:|---|
| 0 | `0x480` |
| 1 | `0x4C0` |
| 2 | `0x500` |
| 3 | `0x540` |

### L1 table at `0x380`

| L1 index | Physical frame |
|---:|---|
| 0 | `0x580` |
| 1 | not present |
| 2 | `0x5C0` |
| 3 | `0x600` |

### L1 table at `0x400`

| L1 index | Physical frame |
|---:|---|
| 0 | `0x640` |
| 1 | `0x680` |
| 2 | `0x6C0` |
| 3 | `0x700` |

### L1 table at `0x440`

| L1 index | Physical frame |
|---:|---|
| 0 | `0x740` |
| 1 | `0x780` |
| 2 | `0x7C0` |
| 3 | `0x800` |

---

# Exercise 1 — Walk the existing page tables

Starting from:

```text
Virtual address = 0x2B7
```

work out the complete path:

```text
                 virtual address
                       │
                       ▼
                    L3[?]
                       │
                       ▼
                    L2[?]
                       │
                       ▼
                    L1[?]
                       │
                       ▼
                  frame 0x???
                       │
                       │ + offset
                       ▼
                 physical 0x???
```

Write down:

- L3 index
- L2 index
- L1 index
- offset
- physical frame reached
- final physical address

---

# Exercise 2 — Determine the target physical frame

The target physical address is:

```text
0x5B7
```

### A.

Which physical frame contains `0x5B7`?

Also determine its offset within that frame.

---

# Exercise 3 — Determine the target virtual page

The target virtual address is:

```text
0x2B7
```

### B.

Which virtual page contains `0x2B7`?

What is its offset within that page?

---

# Exercise 4 — Find the page-table indexes

### C.

Split `0x2B7` into:

```text
L3 index = ?
L2 index = ?
L1 index = ?
offset   = ?
```

Use the address layout:

```text
┌────────┬────────┬────────┬────────────┐
│   L3   │   L2   │   L1   │   Offset   │
│ 2 bits │ 2 bits │ 2 bits │   6 bits   │
└────────┴────────┴────────┴────────────┘
   11-10     9-8      7-6       5-0
```

---

# Exercise 5 — Walk from CR3

Start from:

```text
CR3 = 0x100
```

### D.

Using your L3 index, determine which L2 table the CPU enters.

Then use your L2 index to determine which L1 table it enters.

Then use your L1 index to determine which physical frame it reaches.

Draw the complete path:

```text
CR3
 ↓
L3[?]
 ↓
physical 0x???
 ↓
L2[?]
 ↓
physical 0x???
 ↓
L1[?]
 ↓
physical frame 0x???
```

---

# Exercise 6 — Is the mapping already present?

### E.

Does the required virtual → physical mapping already exist all the way down to an L1 entry?

Compare the physical frame you need from Exercise 2 with the physical frame reached in Exercise 5.

If they are different, identify where the existing mapping fails to produce the desired target.

---

# Exercise 7 — Imagine the mapping does not exist

Now imagine that the page-table path required for the target mapping does not exist.

The OS has the following free physical frames:

```text
Free frames:

0x840
0x880
0x8C0
0x900
```

The OS follows this rule:

> Whenever it needs to create a new page table, allocate the first free frame.

Remember:

> A page table itself occupies one physical page/frame.

### F.

At which level would the OS need to create a new page table?

### G.

Which frame would the frame allocator give to that new page table?

### H.

What would the parent page-table entry need to contain after the new page table is created?

### I.

What would the final L1 entry need to contain to establish:

```text
target virtual page
        ↓
target physical frame
```

---

# Exercise 8 — Connecting this to Rust's `physical_memory_offset`

Now introduce one of the important concepts from Phil Opp's tutorial.

The kernel cannot simply dereference a physical address.

The bootloader has instead provided:

```text
physical_memory_offset = 0x8000
```

The rule is:

```text
virtual address = physical address + 0x8000
```

So:

```text
physical address X
        ↓
virtual address X + 0x8000
```

### J.

The L3 page table is physically located at:

```text
0x100
```

What virtual address should Rust use to access it?

### K.

The L2 page table is physically located at:

```text
0x180
```

What virtual address should Rust use to access it?

### L.

In general, write the formula that converts a physical address into the virtual address the kernel can use to access it.

---

# Final mental model

Once you've completed the exercises, try to explain this diagram in your own words:

```text
                    CPU
                     │
             virtual address
                     │
                     ▼
              ┌─────────────┐
              │     L3      │
              │    table    │
              └──────┬──────┘
                     │
                  L3[index]
                     │
                     ▼
              ┌─────────────┐
              │     L2      │
              │    table    │
              └──────┬──────┘
                     │
                  L2[index]
                     │
                     ▼
              ┌─────────────┐
              │     L1      │
              │    table    │
              └──────┬──────┘
                     │
                  L1[index]
                     │
                     ▼
              physical frame
                     │
                + offset
                     │
                     ▼
              physical address
```

And alongside it:

```text
OS / Rust
   │
   │ needs to inspect or modify
   ▼
physical page tables
   │
   │ physical_memory_offset
   ▼
virtual addresses
   │
   ▼
Rust references / pointers
```

## What this exercise is preparing you to understand

The toy exercise corresponds conceptually to the pieces in Phil Opp's implementation:

| Exercise concept | Paging implementation concept |
|---|---|
| Root table | `CR3` / active level-4 table |
| L3/L2/L1 tables | `PageTable` hierarchy |
| Virtual address split | `Page` / page-table indexes |
| Physical frame | `PhysFrame` |
| Virtual page | `Page` |
| Page-table entry | `PageTableEntry` |
| Creating a mapping | `Mapper::map` |
| Finding a free frame | `FrameAllocator` |
| Physical → kernel virtual address | `physical_memory_offset` |
| Walking tables in software | `translate_addr` / mapper logic |

The key question to keep in mind while reading the Rust is:

> **"What physical page-table operation is this line of Rust performing?"**

That question is often more useful than trying to understand the Rust abstraction first.
