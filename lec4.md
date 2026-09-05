# embedded C
## Register Macros

```c
#define SET_BIT(REG, BIT)     ((REG) |= (1U << (BIT)))
#define CLEAR_BIT(REG, BIT)   ((REG) &= ~(1U << (BIT)))
#define GET_BIT(REG, BIT)     (((REG) >> (BIT)) & 1U)
```

## Loop & Setup Structure

```c
int main(void)
{
    /* --- Setup (runs once) --- */
    SET_BIT(RCC_AHB1ENR, 0);        // Enable GPIOA clock
    GPIOA_MODER |= (1U << 10);      // Set PA5 as output (MODER5[1:0] = 01)

    while (1)                        // "Super loop"
    {
        TOGGLE_BIT(GPIOA_ODR, 5);   // Toggle PA5
        for (volatile int i = 0; i < 500000; i++);  // crude delay
    }
}
```
The `while(1)` is called the **super loop** 

## File Guard

```c
#ifndef GPIO_DRIVER_H
#define GPIO_DRIVER_H

/* declarations here */

#endif /* GPIO_DRIVER_H */
```
Prevents a header from being included more than once in the same translation unit — without it, if two files both `#include` a header that itself gets `#include`d again indirectly, you get duplicate declarations/definitions and a compile error.



# Compilation and tool chain

>preprocessor->compiler->assembler->linker


## Preprocessor

The preprocessor is the first stage of the build pipeline, running before the compiler proper. It performs pure text transformation on the source file — no syntax or semantic checking occurs here — and produces a single, expanded **translation unit** ready for compilation.

**Core responsibilities:**

- **File inclusion (`#include`)** — Recursively inlines header file contents at the point of inclusion. `#include <file.h>` searches system/toolchain include paths (e.g. CMSIS or arm-none-eabi-gcc's default include dirs); `#include "file.h"` searches the local project directory first.

- **Macro expansion (`#define`)** — Substitutes object-like macros (`#define MAX_BUF 64`) and function-like macros (`#define SET_BIT(reg, bit) ((reg) |= (1U << (bit)))`) textually, wherever they're referenced. <!-- Includes **token pasting** (`##`) and **stringizing** (`#`) operators for more advanced macro construction. -->

- **Conditional compilation (`#ifdef`, `#ifndef`, `#if`, `#elif`, `#else`, `#endif`)** — Includes or excludes blocks of code based on whether a macro is defined or an expression evaluates true. Commonly used for:
  - **Include guards** (`#ifndef FILE_H` / `#define FILE_H` / `#endif`) to prevent duplicate header inclusion — though `#pragma once` is a common non-standard alternative most compilers (including GCC) support.
  - Platform/target-specific code paths (e.g. compiling different register definitions per MCU variant).
- **Comment removal** — Strips `//` and `/* */` comments entirely; they never reach the compiler.
- **Line control** — Tracks `#line` directives and file/line mapping so compiler diagnostics still point to the original source location despite the expansion.

**Output:** a single flattened, macro-expanded **preprocessed source file** (`.i` for C, `.ii` for C++), containing no directives, no comments, and no `#include`s — just plain C ready for lexing and parsing by the compiler front end.

<!--
**Inspecting it directly:** with the ARM GCC toolchain, you can stop the pipeline at this stage to see exactly what the compiler will see:

```bash
arm-none-eabi-gcc -E main.c -o main.i
```


This is invaluable for debugging macro-related bugs (e.g. an unexpected value from a function-like macro, or a header being included multiple times without a guard) since the resulting `.i` file shows the fully expanded, ground-truth source.
-->

## compiler

The compiler takes the preprocessed source (`.i`) and translates it into target-specific **assembly code**, while performing all the checks the preprocessor skipped. This is the stage where C syntax and semantics actually matter.

**Core responsibilities:**
<!--
- **Lexical analysis (tokenizing)** — Breaks the flat character stream into tokens: keywords, identifiers, operators, literals, punctuation.
- **Syntax analysis (parsing)** — Builds an **Abstract Syntax Tree (AST)** from the token stream according to C grammar rules. This is where malformed syntax (missing semicolons, mismatched braces, invalid expressions) is caught.
- **Semantic analysis** — Type checking, scope resolution, and validation that the code is logically consistent: mismatched types, undeclared identifiers, invalid pointer conversions, incompatible function signatures, etc. Most `-Wall -Wextra` warnings surface here.
- **Intermediate Representation (IR) generation** — GCC lowers the AST into an internal IR (GIMPLE, then RTL) that's architecture-agnostic, enabling optimization passes independent of the final target.
- **Optimization** — Applied to the IR based on the `-O` level (`-O0` through `-O3`, `-Os`, `-Og`). Includes dead code elimination, constant folding, loop unrolling, inlining, register allocation hints, etc. Higher optimization can reorder or eliminate code in ways that matter for embedded work — e.g. a `volatile`-qualified register access is required precisely to prevent the optimizer from eliminating or reordering hardware register reads/writes.
- **Code generation** — Lowers the optimized IR into **target-specific assembly**, honoring the architecture flags you pass in (for the F401CC's Cortex-M4F: `-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard`). This determines instruction selection (Thumb-2 encoding), register usage conventions (AAPCS), and whether FPU instructions are emitted.
-->
- **lexical analysis (tokenizing)** - transforms the large code into tokens of keywords , identifier and operators from the programming language
- **Syntax analysis (parsing)** - if there is a syntax error based on the language grammer it's caught here
- **Code generation** - generates an assembly code based on the flags you determine *(very important mentioned in the lecture)* like if the floating point unit is used 

**Output:** a target-specific **assembly file** (`.s`), still human-readable, not yet machine code, and not yet resolved to real memory addresses.

## Assembler

The assembler takes the compiler's output — target-specific assembly (`.s`) — and translates it into **machine code**, producing a relocatable object file. This is a much more mechanical, one-to-one translation than compilation: each assembly mnemonic maps to a specific binary instruction encoding, with no semantic analysis or optimization involved.

**Core responsibilities:**

- **Instruction encoding** — Converts human-readable mnemonics (`MOV`, `LDR`, `STR`, `BL`, etc.) into their binary machine-code representation, per the target's instruction set encoding

- **Symbol table construction** — Records every label, function name, and global/external symbol defined in the file and makes an offset table of it to be uploaded to the real memory addesses after

- **Code generation** —  assembler generates a machine language (`.o`)

**Output:** a **relocatable object file** (`.o`), in ELF format

### Note (very important)
Both the **compiler** and **assembler** process each source file **individually**  When a file references an `extern` variable, an external function, or any symbol not defined in that same file, the compiler/assembler cannot know its final address, so they leave a **placeholder**.

## Linker

The linker combines all relocatable object files (`.o`) and static libraries (`.a`) into one fully-resolved executable, resolving every symbol reference to a real memory address.

**Goals:**
- Resolve every symbol globally — match each placeholder left by the compiler/assembler to its actual definition across all inputs
- Merge like sections (`.text`, `.data`, `.bss`, etc.) from all object files into contiguous regions
- Assign final memory addresses per the **linker script**- a script determine how the linker should work- (`.ld`) — mapping sections onto real FLASH/RAM regions
- Patch every relocation entry with the now-known final address
- Set the initial address of the program mapped memory which contains the stack pointer address at first then IVT (interrupt vector table) 

**Errors it catches:**
- **Undefined reference** — a symbol is used but never defined anywhere in the inputs
- **Multiple definition** — the same symbol defined in more than one object file
- **Section overflow** —  e.g. code too large for FLASH

**Output:** a linked ELF and a  `.bin`/`.hex` for flashing.


# simulator vs emulator 
* **Simulator:** Simulates the MCU's behavior at a higher level, mainly letting you observe peripheral behavior and outputs without fully reproducing the MCU internally.
* **Emulator:** Reproduces the MCU's internal operation more closely, allowing you to inspect and manipulate registers, memory, CPU state, and peripheral states while running the actual code.


# Uploading Process:
* **USB-to-CAN:** Laptop → USB-to-CAN adapter → CAN bus → MCU(contains the bootloader)
* **MCU-to-MCU:** Laptop → Programmer/MCU(contains the boatloader) → Target MCU

# cross compiler vs native compiler 
* **Cross compiler:** Compiles code on one system for a **different target system/architecture**.
* **Native compiler:** Compiles code for the **same system/environment on which the compiler is running**.



# Startup Code

initilizes the following:
**1. Vector table (data, not executable code)**
- A fixed array of 32-bit words placed at the base of flash (`0x00000000`)
- Word 0: initial Main Stack Pointer value — loaded directly into MSP by hardware, no code runs for this
- Word 1: address of `Reset_Handler` — the reset vector whether it's the boatloader or startup routine
- Remaining words: addresses of fault handlers, system exceptions, and every peripheral interrupt handler, in the fixed order

**2. `Reset_Handler` (code — the actual startup routine)**
this is the code being executed directly after the stack pointer is initiliazed it can be **bootloader** that leads to startup routine
or it can be **startup routine**
Runs once hardware loads its address into the PC. Responsibilities, in order:
1. Copy `.data` from FLASH to RAM
2. Zero `.bss` (uninitialized globals must start at 0 per the C standard)
3. Call `main()`

**3. Default/weak exception & interrupt handlers**
Stub functions (typically infinite loops) for every vector table entry, marked `.weak` so application code can override any one (e.g. `TIM2_IRQHandler`) by simply redefining a function with the same name elsewhere — without editing the startup file.


# Bootloader

## Overview

Siemens-methodology for bootloader follows primary/secondary bootloader formulation  . On reset, the program counter points to the primary bootloader, which checks whether an update signal indicates new code needs to be uploaded or not.

## Primary Bootloader

- Runs first on every reset — the PC starts at the primary bootloader (second word in the vector table) after initilizing the stack pointer
- Checks for an update-request signal to determine whether new code is being uploaded
- If no update is requested, it proceeds to call the startup routine 
- If an update is requested, it loads the secondary bootloader into RAM


## Secondary Bootloader

- Loaded into the MCU (into RAM) by the primary bootloader when an update is requested
- Responsible for the actual flash write: erasing the existing program and writing the new code into ROM/flash in a determined program memory address by the programmers from before
- Temporary by nature — it exists only for the duration of the update and is erased once the MCU restarts (as it was loaded into RAM-volatile memory-) into the newly flashed program

## Upload Process

1. An upload program on the PC sends a restart signal to the MCU
2. The MCU restarts and re-enters the bootloader's startup code
3. The primary bootloader checks for the update-request signal
4. If present, the secondary bootloader is uploaded into RAM
5. The secondary bootloader erases the flash program part (leaving the startup routine and bootloader) and programs it with the new code
6. Once uploading is complete, the MCU is restarted
7. On this restart, the newly flashed program runs, and the secondary bootloader (having lived only in RAM) is gone


# Makefile

* If you have multiple source files in c and want to compile them from Terminal Command, it is hard to write command to compile all this files every time.

* To solve such kind of problem, we use Makefile because during the compilation of large project we need to write numbers of source files as well as linker flags are required, that are not so easy to write again and again.

* Makefile is a tool to simplify or to organize code for compilation.

* Makefile is a set of commands (similar to terminal commands) with variable names and targets to create object file and to remove them. In a single make file we can create multiple targets to compile and to remove object, binary files. You can compile your project (program) any number of times by using Makefile.Contain


# pooling vs interupt
* **Polling:** The CPU continuously checks a device or flag to detect whether an event has occurred.

* **Interrupts:** The device notifies the CPU when an event occurs, allowing the CPU to execute the corresponding ISR.

* **Atomic Instructions:** Execute as one indivisible operation and cannot be interrupted or preempted midway through execution.

## non maskable interrupts vs maskable interruptss

* **Non-maskable interrupts (NMI):** Cannot be disabled or ignored by normal interrupt-masking mechanisms and are reserved for critical events such as reset.

* **Maskable interrupts:** Can be disabled or ignored by the CPU using interrupt-masking mechanisms when necessary.

* **Masking mechanisms:** Methods to disable or suppress maskable interrupts, such as the **interrupt enable/disable flag** and **interrupt mask registers**.



## critical section 
A section of code that must execute without interruption because it accesses a shared critical resource (os term you can search it); and this is done by disabling/masking interrupts before entering and restoring the interrupt state after leaving


## re enterant vs enterant functions
* **Re-entrant Function:** Can be safely interrupted and called again before a previous execution finishes without corrupting its state.

* **Non-re-entrant Function:** Cannot be safely called again while already executing because it may share or modify unsafe state.


## Interrupt Time 

- An interrupt doesn't execute instantly — there's a measurable delay between the interrupt signal occurring and the CPU actually responding to it
- Normal program execution (**Background**) is suspended when an interrupt occurs, and resumes only after the ISR fully completes
- Before running any ISR code, the CPU must save its current context (register state) so it can resume Background execution correctly afterward — this save isn't free, it takes time
- Symmetrically, after the ISR finishes, the CPU must restore that saved context before returning to Background — also not instantaneous
- **Interrupt Latency** = the delay between the interrupt occurring and the CPU beginning to respond (starting the context save) — this is the "reaction time" of the system
- **Interrupt Response** = the total delay between the interrupt occurring and the actual ISR code starting to run — this is latency *plus* the context-save time, so it's always ≥ latency
- **Interrupt Recovery** = the time taken after the ISR code finishes to restore context and get back to Background — the "cleanup cost" of returning from an interrupt
- The full cost of handling an interrupt isn't just "how long the ISR code takes" — it includes the save/restore overhead on both sides, which is why latency, response, and recovery are tracked as distinct metrics
- This breakdown matters for real-time systems design: worst-case interrupt latency/response determines whether a system can meet its timing deadlines


## SYNCHRONOUS vs ASYNCHRONOUS API


**SYNCHRONOUS**
*   You are in a queue to get a movie ticket. You cannot get one until everybody in front of you gets one, and the same applies to the people queued behind you.
*   Blocks caller till operation done by functions is completed. Caller is blocked till function returns at the operation end.
*   SYNCHRONOUS EXAMPLE. Any process consisting of multiple tasks where the tasks must be executed in sequence.

 **ASYNCHRONOUS**
*   You are in a restaurant with many other people. You order your food. Other people can also order their food, they don't have to wait for your food to be cooked and served to you before they can order. In the kitchen restaurant workers are continuously cooking, serving, and taking orders. People will get their food served as soon as it is cooked.
*   Functions starts operation requested by caller. function return does not indicate operation finish. Caller will be notified later when the operation finishes.


---

### Updates & Materials

All upcoming lectures, course updates, and additional materials will be continuously published to this
**[GitHub repository](https://github.com/eng-Aly/siemens-academy-embedded-systems-course.git)**
