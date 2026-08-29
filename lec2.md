# **Revision**
# CPU arch.
- ALU
- Control Unit
- GPRs (General Purpose Registers)
- SRs  (Status Registers)

## GPRs (General Purpose Registers)
high-speed internal CPU storage locations used to hold data, memory addresses, and intermediate results during program executio

## Status Registers
registers that store current state or results **indicators** of operations 

# Instruction Life Cycle
fetch->Decode->Execute->(Memory)

---

# **Lec2**

# Pipelining 
simply doing an instruction while other one is being executed like while fetching (I/O Operation no CPU needed) you can decode other instruction

> Pipelining is done only for **RISC** (reduced instruction set computers) processors and can't be done for **CISC** (complex instruction set)

## control hazards in pipelining
A control hazard in a computer pipeline happens when the processor cannot decide which instruction to fetch next because a branch or jump instruction has not finished yet



```text
like if you have an if condition decoding its instruction 
you won't know what is the next instruction to handle 
```
### control hazards solutions
- **stalling**: freeze the pipeline till the instruction result is determined through **bubbles** or **nop**(if you saw it before in a cpu halt func for example) instructions
- **branch prediction**: predicition through hardware algos which one will get executed

## data hazards in pipelining
Data hazards in pipelining occur when an instruction depends on the result of a previous instruction that is still moving through the pipeline



```text
an instruction depends on the result of the instruction before so fetching & decoding
before the other one writes to memory leads to an un-expected value
```
### data hazards solutions 
- **stalling**: freeze the pipeline
- **Forwarding**:skipping or bypassing instructions that depend on each other to not sabotage outputs or data

# watchdog timer
an electronic or software timer that automatically resets a computer or microcontroller if the system freezes or crashes
must be **kicked** (triggered or restarted) periodically so it doesnt restart the MCU
> can be turned on in boot seq or in runtime

# MCUs arch.
communication between different CPU modules (program memory , data memory , peripherals) through **data** , **addresses** and **control** buses

>MCUS differ based on how they are connected to 1-**program memory** 2-**peripherals belong to data memory or not** 

## Harvard arch.
communication between **CPU** , **Program memory** and **Data memory** is done on **seperate busses**

### Problem: Separate Address Spaces

Program memory and data memory have **independent address spaces**.

Therefore, both memories can have an address such as:

```text
Program Memory:  0x000000
Data Memory:     0x000000
```

This **does not cause actual confusion for the CPU**, because the bus/interface used for the access determines which memory is being addressed.

However, it can make **programming and system design less intuitive**, because the same numeric address may refer to different locations depending on whether the CPU is performing an **instruction fetch** or a **data access**.

## Von neumen arch.
communication between **CPU** , **Program memory** and **Data memory** is done on **the same bus**  e.g. **AVR ATmega328P**

### Problem: common interface
common bus between program memory and data memory may hinder **pipelining** or **multi-tasking**

> solution was hybrid arch. not in the lecture context but worth mentioning 

> peripherals can be accessed through memory mapped IO and port-mapped IO not in the lecture context but worth mentioning

# CISC vs RISC


**CISC (Complex Instruction Set Computer)** and **RISC (Reduced Instruction Set Computer)** are two approaches to designing a CPU's instruction set and execution architecture.
| Feature                    | CISC                                                                              | RISC                                                               |
| -------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Instruction set**        | Large and complex                                                                 | Small and simple                                                   |
| **Instruction length**     | Usually variable                                                                  | Usually fixed or limited formats                                   |
| **Instruction complexity** | An instruction may perform several operations                                     | Instructions usually perform a single/simple operation             |
| **Memory access**          | instructions can directly access memory                                           | Normally only **load/store** instructions access memory            |
| **Instruction execution**  | More variable; some instructions may require many cycles                          | More uniform/predictable                                           |
| **Control unit**           | Often more complex can't be used in MCUs                                          | Usually simpler used in MCUs                                                  |
| **Compiler dependence**    | Less work                                                                         | More work                                                          |
| **Hardware complexity**    | Higher                                                                            | Lower                                                              |
| **Pipeline design**        | can't be done genrally                                                            | Generally easier                                                   |
| **Code size**              | Often smaller                                                                     | Can be larger                                                      |



# memory types

* **Program memory (ROM/Flash):** **non-volatile memory** used to store the program and other persistent data. During normal runtime, it is **typically read by the CPU**, but it is **not inherently read-only**. 
  * Flash can be **programmed/erased** using specific procedures or interfaces, such as **DFU, SWD, bootloader commands, or a dedicated flash controller**.
  * Flash normally **cannot be written like RAM to an arbitrary byte/address**. but can be done only to a sector or a page as a whole of the flash
            - **.text:** executable program instructions.
            - **.rodata:** read-only constants **global constants**.
            - **.data:** **initialized global** , **static** variables; their initial values are stored in Flash and copied to RAM at startup.
            - **.bss:** uninitialized or zero-initialized global/static variables; allocated in RAM and initialized to zero at startup.
            - **IVT:** interrupt vector table , interrupt functions addresses 
            - **boot:** used for bootloader or boot-related code/data (not-standard)
  * there are **OTP** one time programmable program memories that are can be programmed once but much cheaper
  ### linker script
  defines the memory layout of the program and tells the linker where to place different program sections in memories such as Flash and RAM.
  
  


* **EEPROM:** **non-volatile memory** used to store your **.env** data , **configs** , **constants** or generally data that can't be lost when the MCU is powered off
    * generally byte-addressable through a certain comm. protocol like **SPI** or **UART**

* **RAM:**  **volatile memory** used to store variables during run time that are **not** **static** or **global**

- **.data:** initialized global/static variables. Their initial values are stored in Flash and copied to RAM during startup.
- **.bss:** uninitialized or zero-initialized global/static variables. Allocated in RAM and initialized to zero during startup.
- **Heap:** dynamically allocated memory (`malloc`, `new`). Usually grows toward higher addresses.
- **Stack:** function-call storage, including local variables, parameters, return information, and saved registers. Usually grows toward lower addresses.


### Startup Code

**Startup code** is the code executed after CPU reset and before `main()`. It prepares the system so that C/C++ code can execute correctly.

Main responsibilities:

- **Initialize the stack:** set the stack pointer to the initial stack location.
- **Reset handling:** execute the reset handler specified by the reset vector.
- **Initialize `.data`:** copy initialized global/static variables from their initial values in Flash to their runtime locations in RAM.
- **Clear `.bss`:** set uninitialized/zero-initialized global/static variables in RAM to zero.


> **Linker script:** defines the memory layout and addresses of sections.
>
> **Startup code:** performs the runtime initialization using those addresses.


### context switching 
saving CPU state at current instruction before **jumping** to another inst.
#### what is saved
* **Program Counter (PC):** address of the next instruction to execute.
* **Stack Pointer (SP):** current stack position.
* **General-purpose CPU registers**
* **Status/flags register**
* **Other CPU-specific registers** when required.
#### where it's saved
in stack generally

---

### Updates & Materials

All upcoming lectures, course updates, and additional materials will be continuously published to this
**[GitHub repository](https://github.com/eng-Aly/siemens-academy-embedded-systems-course.git)**
