---
layout: post
title: 'How does a C program run on a Unix system'
date: 2024-04-14
categories: dev
---

To understand how a program is run on Unix system (such as Linux or macOS), we will cover three topics:

- Compilation of the source code into an executable file
- Organization of the hardware
- Execution of the executable file

The source code we will use is a simple "hello, world" program written in C:

```c
#include <stdio.h>
int main()
{
	printf("hello, world");
	return 0;
}
```

## Create an executable file

![Compile Steps](/assets/images/2024/04/compile-steps.png)

As we can see in the diagram, the `hello` source file, when process with `GCC` compiler, will go through four phases to become an executable binary file:

- Preprocessor phase: Include the header file, remove comments...
- Compiler phase: Check syntax errors, translate the code to assembly languages.
- Assembler phase: Translate the code into binary, created a file called "relocatable object program". This program can be used to link with other programs, such as the `printf` program.
- Linker phase: Include the implementation of the dependent libraries and programs into one unified binary file. The different between linker and the preprocessor is that the preprocessor includes only type declarations from libraries, while the linker includes the actual implementations of the libraries.

At this point we have a single "hello" binary file, which can be run in the shell using a command like this:

```
./hello
```

## Hardware organization

Before running the `hello` program, let's have an overview of hardware organization.

<Image src={hardwareOrganization} alt="" />
![Hardware organization](/assets/images/2024/04/hardware-organization.png)

**Buses**

Collection of electrical conduits carry bytes of information between components. Bytes are carried in fixed chunk sizes called "word", a word consists of either 4 bytes (32-bit system) or 8 bytes (64-bit system).
Data is transferred between components via buses. There is a central I/O bridge, which can:

- transfer data to and from the CPU via the System Bus.
- Transfer data to and from the Main Memory via the Memory Bus.
- Transfer data to and from I/O devices via the I/O Bus.

**I/O devices**

There are four main I/O devices:

- the user keyboard for input
- the user mouse for input
- the display for output
- the disk for long-term storage of data and programs
  Data will be transfer from I/O devices to other components, such as Main Memory or CPU, using Buses.

**Main Memory**

Main Memory is a temporary storage device that holds the program code and the data that the processor manipulates when executing the program.
Physically, main memory contains a collection of dynamic random access memory (DRAM) chips.
Logically, main memory consists of a linear array of bytes, with each byte having a unique address. This means that we can access the data using an index.

**Processor (a.k.a central processing unit - CPU)**

In the CPU, we have:

- Program Counter (special register): Stores the pointers that point to the addresses of the instruction code in the main memory. Thanks to the Program Counter, the instructions will be executed sequently
- Register File: Stores the word-size general-purpose registers. Each register stores the data that is fetched from the main memory or stores the result from ALU calculations.
- Arithmetic/Logic Unit (ALU): Calculates inputs from registers and stores outputs into a register.

## Running the program

Now we have an executable binary file and an overview of hardware organization. It's time to run the `hello` program.
To run the program, type the application name in the `shell`. This will display the `hello, world` in the screen.

```
./hello
hello, world
```

Let's see how the program is run:

1. Initially, the shell program will be waiting for user input
2. User types "./hello" and hits enter, which means asking the system to run the `hello` execution file
3. The `hello` code is copied from the Disk to the Main Memory
4. There is one instruction in the `hello` file (the `printf("hello, world")` instruction). Its address in the Main Memory will be stored in the Program Counter
5. The data (`hello, world`) is copied from the Main Memory to the Register in the Register File
6. In the `hello` program, we don't have any calculations, so there is no need to use the `ALU`. The `hello, world` string will be copied from the Register to the Display.
7. The user will see that the `hello, world` is printed on the screen
8. The program terminates, and the `shell` will wait for the next program to be executed based on user input.

That's how the simple `hello` program is run. Thanks for reading until this far!

Images and content are referenced from the book [Computer Systems: A Programmer's Perspective](https://www.amazon.com/Computer-Systems-Programmers-Perspective-3rd/dp/013409266X)
