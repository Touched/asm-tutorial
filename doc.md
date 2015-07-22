# Assembler for the GBA

## Chapter 1: Memory

## Chapter 2: The CPU

### Architecture
The CPU in the GBA is called the ARM7TDMI. ARM7 refers to the series of chips and TDMI are names given to the features supported by this chip. For example, the 'T' indicates that this chip supports THUMB mode. The ARM7TDMI implements the ARMv4 instruction set and supports two versions of the instruction set. The 32 bit ARM instruction set and the 16 bit THUMB instruction set. An instruction set is the list of operations that a CPU can perform. Each instruction is represented by an *opcode* which has various *operands* which control the functioning of the opcode. Opcodes are difficult for humans to understand and thus are mapped to letter sequences called *mnemonics* which are easier for humans to remember, read and write. THUMB is often the preferred instruction set, as the GBA Gamepak only has a 16 bit address bus, this means only 16 bits can be read in one go from the ROM - this makes THUMB execute faster.

### Registers
Registers are small data containers which can be directly accessed and manipulated by the CPU at high speed. In order to manipulate data, a CPU must load data from memory into a register. Once in a register, data can be freely manipulated and then compared, or it may be stored in memory again to be accessed later. In the ARM7TDMI, registers are numbered `r0` through `r15`, the last 3 typically having special meaning. Each register is 32 bits or 4 bytes in size. The registered are additionally divided,`r8` through `r15` being *high registers* and `r0` through `r7` being *low registers*. THUMB mode is mostly limited to using the low registers; high registers can only be handled by a select few opcodes.

### Words and Data Sizes
A *word* is the name given the to the unit of data that the CPU process in one go, i.e. the size of a register. On the ARM7TDMI, we have a 32 bit word. A Halfword is thus 16 bits. In addition to this, we have the byte, which is 8 bits.

### Special Registers
We have three special registers in THUMB:
1. `r13` - Stack Pointer (`SP`)
2. `r14` - Link Register (`LR`)
3. `r15` - Program Counter (`PC`)

#### Program Counter
Perhaps the easiest to understand, the Program Counter keeps track of the current instruction to be executed. This value, if read by code, is often a few opcodes ahead due to prefetch. However, if theory it should be the address of the current instruction. PC, like most of the special registers, is rarely modified directly. It is automatically updated by the processor every opcode as well as being modified by branching instructions.

#### Stack Pointer
The stack pointer is the key element with which we access the stack. The stack is an incredibly simple data structure which allows arbitrary data storage. It is most commonly used as a way to 'back up' registers at the start of a subroutine. The stack is often a huge source of confusion for newbies, who have many misconceptions about reasons for its use.

The stack is known as a LIFO (Last in, First out) data structure. This simply means that the last item to be added to the stack is the first item to leave it. Imagine this as a pile of books, when you add a book to the pile, you simply place it on the top. When removing a book, we take the top book off - we can't take something off the bottom lest pile collapse. The topmost book would be the last (most recent ) book we added, while the bottom would have been the first to be added. Thus, the pile is a LIFO structure.

The stack contains words (just like books - look how far my analogy extends!). However, all we know about the stack is the location of the most recent item - this is the stack pointer. The stack pointer must thus be a multiple of 4. This ensures the stack is properly aligned. Alignment is critical to preserve the data on the stack. The reason the stack can be used for generic data storage is because the stack stores data indiscriminately (as long as it is word aligned). The stack does not care where the data came from, or what you do with it, just so long as you keep it aligned. This is a common source of confusion for newbies who assume (due to the syntax of push and pop), that the stack "remembers" what registers you added to it. It does not - it only remembers the values they contain. The order in which you add to and remove from the stack is thus critical. Luckily, push and pop work in a very specific order to minimize to make sure you can use it to preserve register values.
The fact that the stack does not care where the data came from makes it useful for a number of things. We can use it to swap the value of registers without using another. For example,
```
mov r0, #3
mov r1, #1

push {r0}
mov r0, r1
pop {r1} ; not a typo
```
This code swaps the values of `r0` and `r1` simply by putting the value of `r0` on the stack, then copying `r1` onto `r0`. It then removes the value that was on `r0` but places it on `r1` instead. This might be thoroughly confusing, so here is another, step by step explanation. 

1. First we set the values of `r0` and `r1` using `MOV`. This is just setting up the example.
2. Next, we back up the value of r0 on the stack
3. `mov r0, r1` **copies*** r1 onto r0. At this point the original value of r0 is lost. This is half of the swap.
4. Now, we get the original value of r0 back. Instead of restoring it back onto r0 (this would put us back where we started), we pop it onto r1. 
5. The swap is now complete: `r1` is now 3 and `r0` is now 1.

Push and pop are very useful, however they can be a bit confusing since a lot of their operation is hidden. Whenever we push, we simply decrement the stack pointer by 4, and then store the value of the register at the new pointer. Pop is the reverse: we increment the stack pointer by 4 and then read the value at the new pointer. Thus, we have rough equivalents,
```
@ push {r0} equivalent
sub sp, #4
str r0, [sp]

@ pop {r0} equivalent
add sp, #4
ldr r0, [sp]
```

The stack grows downwards by convention, thus we subtract from `SP` when adding to the stack, and add to it when removing. Thus, smaller values of `SP` indicate a larger stack and vice versa.

The stack does have a storage limit - when we run out of space on the stack it is called *Stack Overflow*. This type of error is commonly seen in recursive functions which recurse too many times.

#### Link Register
Link register is used to keep track of the return location when calling subroutines. Whenever a `BL` instruction is encountered, LR is automatically set to PC + 4. Since BL is 4 bytes long (even in THUMB), PC + 4 is the next instruction. When the subroutine is done execution, it returns execution back to the calling routine by setting PC = LR. This is done in a number of ways:

If we have not pushed LR, we can return with the following code,
```
bx lr
```
However, if we had a nested subroutine and needed to save LR on the stack, the following code is often seen,
```
pop {r0}
bx r0
```
This code is used when ARM-THUMB interworking is desired and there is no return value. Since `BX` is the only way to return to ARM code from THUMB code (and vice versa), we must return using this opcode in order to set the correct mode of execution. If there is a return value, then `r1` is simply used instead. 

If ARM-THUMB interworking is not necessary, then we can return with
```
mov pc, lr
```

or 

```
pop {pc}
```

Which are equivalent to the respective snippets above. These instructions will cause the code returned to to be executed in the same mode as the code however.

## Chapter 3: Control Flow

### Functions

### Calling Convention

### Comparisons

### Branches

### Conditional Branching

### CPSR