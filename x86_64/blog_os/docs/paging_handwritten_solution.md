### Target

Virtual address: 0x2B7 (0010 1011 0111)
Physical address: 0x5B7 (0101 1011 0111)

### Exercise 1

Offset bits: 11 0111
L1 Bits: 10
L2 Bits: 10
L3 Bits: 00

L3 Index: 0
L2 Index: 2
L1 Index: 2
Ultimately value of L1[2] should be 0x580

So physical frame reached: 0x580
Offset: 0x37
Final Physical address = Physical Frame reached + Offset = 0x580 + 0x37 = 0x587

### Exercise 2

Frame: 0x580
Offset: 0x37
Physical address = Frame + Offset = 0x580 + 0x37 = 0x587

### Exercise 3

Virtual page containing 0x2B7 = 0x280
Offset within page = 0x2B7 - 0x280 = 0x37 (11 0111)

### Exercise 4

Offset bits: 11 0111

L1 Bits: 10
L2 Bits: 10
L3 Bits: 00

L3 Index: 0
L2 Index: 2
L1 Index: 2

### Exercise 5

CR3 -> 0x100 --> This means that L3 is present at this physical frame address
L3[0] -> Not Present, which means can't go to L2 and L1 pages and calculate physical frames

### Exercise 6

Since L3[0] page table index is not present, so the calculation of page table index L2[2] and L1[2] can't be done
What required is
L3[0] : should be a present and physical frame address pointing to L2 page table
L2[2] : should be present and physical frame address pointing to L1 page table
L1[2] : should be present and physical frame address pointing to physical frame address of 0x580

### Exercise 7

Ques. At which level would the OS need to create a new page table?
Ans. Since L3[0] page table entry is missing, this would imply that all the below page tables will need to be allocated, which are L2[2] and L1[2]

Ques. Which frame would the frame allocator give to that new page table?
Ans. As per the rule

> Whenever it needs to create a new page table, allocate the first free frame.
> Let allocate frame at address 0x840 to L3[0] which means

L3 Table

| L3 Index | Physical Frame |
| -------: | -------------- |
|        0 | `0x840`        |
|        1 | not present    |
|        2 | not present    |
|        3 | not present    |

Since L2 table is also a new table lets allocate next free frame at address 0x880
L2 Table

| L2 Index | Physical Frame |
| -------: | -------------- |
|        0 | not present    |
|        1 | not present    |
|        2 | `0x880`        |
|        3 | not present    |

Now L1[2] entry show point to page frame where physical address of 0x587 should lie
L1 Table

| L1 Index | Physical Frame |
| -------: | -------------- |
|        0 | not present    |
|        1 | not present    |
|        2 | `0x580`        |
|        3 | not present    |

### Exercise 8

Not 100% sure if these question are that simple to answer, since each virtual address need to go through the address conversion from L3 -> L2 -> L1 -> Frame address + offset

Ques J. What virtual address should Rust use to access to access 0x100
Ans. virtual address = 0x100 + 0x8000 = 0x8100

Ques K. What virtual address should Rust use to access 0x180?
Ans. virtual address = 0x180 + 0x8000 = 0x8180

Ques L. General formula
Ans. Virtual address = frame_address + offset
