
## ==Information is bits==

Our code starts as a source file , Source files are organized in 8bit chucks called Bytes , Each byte represents some text character in the program.

All the information written in our code is then later stored in memory.

>Same sequence of bytes can represent an integer, floating point , string , machine structure. So we got to know how systems interpret those numbers

## ==Program compilation phase==

C statements must be translated by other programs into a sequence of low-level machine language instructions. These instructions are then packaged in a form called an executable object program and stored as a binary disk file.

The translation from source file to object file is performed by the compiler driver.
For example, the GCC compiler driver reads the source file hello.c and translates it into an executable object file hello.

The translation is performed in 4 phases ( **Preprocessor , compiler , assembler , linker** )

![[Pasted image 20260406182255.png]]

**Preprocessor Phase**: The preprocessor (cpp) modifies the original C program according to directives that begin with the ‘#’ character. For-example, the \#include command in line1 of `hello.c` tells the preprocessor to read the contents of the system header file `stdio.h` and insert it directly into the program text. The result is another C program , typically with the .i suffix.

**Compiler phase**: The compiler (cc1) translates the text file `hello.i` into the text file `hello.s` , which contains an assembly language program (assembly representation of the original program).

**Assembly phase**: the assembler (as) translates `hello.s` into machine language instructions , packages them in a form known as a relocatable object program , and stores the result in the object file `hello.o` 

**Linkage phase**: Notice that our hello program calls the printf function, which is part of the standard C library provided by every C compiler. The printf function resides in a separate precompiled object file called `printf.o`, which must some how be merged with our `hello.o` program. The linker (ld) handles this merging. The result is a ready to use executable.

## ==**Understanding the compilation phase**==

We can rely on the compilation system to produce correct and efficient machine code. However, there are some important reasons why programmers need to understand how compilation systems work:

1. **Optimizing program performance**: Modern compilers are sophisticated tools that usually produce good code. However, in order to make good coding decisions in our C programs, we do need a basic understanding of machine-level code and how the compiler translates different C statements into machine code. For example, is a switch statement always more efficient than a sequence of if-else statements? We will later describe how compilers translate different C constructs into this language, tune the performance of your C programs , learn how C compilers store data arrays in memory, and how your C programs can exploit this knowledge to run more efficiently.
   
2. **Understanding link-time errors**: some of the most complex programming errors are linker related error, For example, what does it mean when the linker reports that it cannot resolve a reference? What is the difference between a static variable and a global variable? What happens if you define two global variables in different C files with the same name? All covered in chapter 7

3. **Avoiding security holes**: For many years, buffer overflow vulnerabilities have accounted for many of the security holes in network and Internet servers. These vulnerabilities exist because too few programmers understand the need to carefully restrict the quantity and forms of data they accept from untrusted sources. We cover the stack discipline and buffer overflow vulnerabilities in Chapter 3 as part of our study of assembly language. We will also learn about methods that can be used by the programmer , compiler , and operating system to reduce the threat of attack.

## ==**Hardware Organization of a System**==

```c
#include <stdio.h>

int main(){
	printf("Hello world\n");
	return 0;
}
```

At this point, our `hello.c` source program has been translated by the compilation system into an executable object file called hello that is stored on disk.

To understand what happens to our hello program when we run it, we need to understand the hardware organization of a typical system:

![[Pasted image 20260409223513.png]]

#### **Buses**: 
Running throughout the system is a collection of electrical conduits called buses that carry bytes of information back and forth between the components. Buses are typically designed to transfer fixed-size chunks of bytes known as words. The number of bytes in a word (the word size) is a fundamental system parameter that varies across systems. Most machines today have word sizes of either 4 bytes (32 bits) or 8 bytes (64 bits).

>Motherboard factories decide the Data buss width which is typically 8 bytes per word. “word” (hardware term) and “word” (data type in programming) are related but not the same thing.

#### **I/O devices**:
Input/output (I/O) devices are the system’s connection to the external world. Each I/O device is connected to the I/O bus by either a controller or an adapter. **Controllers** are chip sets in the device itself or on the system’s main printed circuit board (often called the motherboard). **An adapter** is a card that plugs into a slot on the motherboard. Regardless, the purpose of each is to transfer information back and forth between the I/O bus and an I/O device.

>Chapter 6 has more to say about how I/O devices such as disks work.
>Chapter10, you will learn how to use the Unix I/O interface to access devices from your application programs.

#### **Main Memory**:
The main memory is a temporary storage device that holds both a program and the data it manipulates while the processor is executing the program. Physically, main memory consists of a collection of dynamic random access memory (DRAM) chips. Logically, memory is organized as a linear array of bytes, each with its own unique address (array index) starting at zero.

>Chapter 6 has more to say about how memory technologies such as DRAM chips work, and how they are combined to form main memory

#### **Processor**:
The central processing unit (CPU), or simply processor, is the engine that interprets (or executes) instructions stored in main memory. At its core is a word-size register called the program counter (PC). From the time that power is applied to the system until the time that the power is shutoff, a processor repeatedly executes the instruction pointed at by the program counter and updates the program counter to point to the next instruction.

Executing a single instruction involves multiple steps done by the CPU:
- Read , The processor reads the instruction from memory pointed at by the program counter.
- Interpretation , interprets the bits in the instruction.
- Executing , performs some simple operation dictated by the instruction.
- Updating , updates the PC to point to the next instruction.

#### **Processor affiliates**:
There are only a few of simple operations that a CPU can execute , and they revolve around main memory, the register file, and the arithmetic/logic unit (ALU).

**The register file** is a small storage device that consists of a collection of word-size registers, each with its own unique name.
**The ALU** computes new data and address values , and is responsible for executing simple athematic operations.

#### **Processor Execution**:

Here are some examples of the simple operations that the CPU might carry out at the request of an instruction:

- Load: Copy a byte or a word from main memory into a register, overwriting the previous contents of the register.
- Store: Copy a byte or a word from a register to a location in main memory, overwriting the previous contents of that location
- Operate: Copy the contents of two registers to the ALU, perform an arithmetic operation on the two words, and store the result in a register, overwriting the previous contents of that register.
- Jump: Extract a word from the instruction itself and copy that word into the program counter (PC), overwriting the previous value of the PC

The shell then loads the executable hello file by executing a sequence of instructions that copies the code and data in the hello object file from disk to main memory. The data includes the string of characters "hello, world\n" that will eventually be printed out.

Using a technique known as direct memory access the data travel directly from disk to main memory, without passing through the processor. Once the code and data in the hello object file are loaded into memory, the processor begins executing the machine-language instructions in the hello program’s main routine. These instructions copy the bytes in the hello, world\n string from memory to the register file ,and from there to the display device ,where they are displayed on the screen.

![[Pasted image 20260410001449.png]]