---
theme: apple-basic
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  Talk presented @ Boston Hackers
drawings:
  persist: false
defaults:
  foo: true
transition: slide-left
title: Kernels, Chaos, and Confidentiality
mdc: true
monaco: true
monacoTypesSource: local # or cdn or none
---
# Kernels and Chaos

Lessons in building an OS

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Let's get started <carbon:arrow-right class="inline"/>
  </span>
</div>


---

### Who is August Radjoe?
&rarr; Intern @ RISC-V, Multiparty.org<br>
&rarr; MS CompSci @ BU <br>
&rarr; Dev @ a few places <br>
&rarr; Open Source @ a few repos, eg. Zephyr
--------------------

### There are many attack surfaces on any kernel
(Linux specific)
Attack surfaces in kernels include:

1. **System Calls**: Bugs can cause privilege escalations or leaks.
2. **Device Drivers**: Vulnerabilities allow arbitrary code execution.
3. **Network Protocols**: Flaws in TCP/IP or Wi-Fi stacks enable remote exploits.
4. **File Systems**: Handling bugs lead to unauthorized access or crashes.
5. **Kernel Modules**: Loaded code with vulnerabilities offers attack entry points.
6. **Buffer Overflows**: Inadequate checks enable kernel memory overwrites.
7. **Race Conditions**: Timing issues cause unintended operation sequences.
8. **Integer Overflows**: Arithmetic mismanagement results in overflows or wrong calculations.
   
---

### We will focus on just 3 of them

&rarr; Syscalls
<br>&rarr; Network Protocols
<br>  &rarr; Buffer Overflow

---

### Structure of the talk

1. Basics of a kernel
   1.  Kernels vs OS, Privilege separation, Types of kernels, XNU vs Linux, Boot process, Switching between rings, How a Syscall works, How does a device driver work, Why can buffer overflows happen
2. How to write a kernel
   1. Writing an image file, Installing GRUB,  Writing a bootloader configuration, Writing a kernel with basic files, threads, basic I/O shell, etc.
3. 3 recent CVEs in the Linux kernel
   1. sdv
4. How do we fix this?
---

# Kernel vs. OS
| Feature        | Kernel           | Operating System  |
| -------------- |:----------------:| :----------------:|
| Definition     | Core component of OS that manages system resources and hardware. | Complete package that includes kernel, UI, and utilities for managing computer resources. |
| Functionality  | Handles low-level tasks like memory management, task scheduling, and device control. | Provides a user interface and a set of applications along with the kernel functionalities. |
| Examples       | Linux kernel, Windows NT kernel. | Linux (Ubuntu, Fedora), Windows, macOS. |

We will focus on talking about kernels, OS increases attack surface, but is too wide in scope for this talk.

(next couple slides, and this one, is inspired from and references [pwn.college's video](https://www.youtube.com/watch?v=j0I2AakUAxk&list=PL-ymxv0nOtqowTpJEW4XTiGQYx6iwa6og))

---

### Encapsulating Privileges

&rarr; A kernel has processes, like individual states (MA, AK, CO etc.), and the kernel itself is like a federal government. <br>
&rarr; There is seperation of privileges, using kernel rings, but they differ in implementations <br>
&rarr; Linux uses ring 0 (kernel mode) and ring 3 (userspace) - which raises the questions what about ring 1 and 2 <br>
&rarr; Well, they're not used as often in modern OSes but there are some that do use all 4 (eg. OS/2)

---

### No one right solution
1. **Linux/UNIX-like**: Separates user (Ring 3) and kernel (Ring 0) spaces, uses capabilities for access control.
2. **Windows NT**: Kernel operates in Ring 0, user apps in Ring 3, uses UAC for privilege management.
3. **macOS/XNU**: Combines Mach (microkernel) for messaging and BSD for system calls.
4. **Microkernels (Minix 3, L4)**: Minimal privileged kernel, services in user space.
5. **Exokernels**: Allows direct hardware access, resource management via user libraries.
---

### Types of Kernels

1. **Monolithic Kernels**: Large, single address space. Examples: Linux, FreeBSD.
2. **Microkernels**: Minimalist, essential functions only. Examples: Minix, L4.
3. **Hybrid Kernels**: Combines monolithic and microkernel features. Examples: Windows NT, XNU.
4. **Exokernels**: Direct hardware access, minimal abstractions. Examples: MIT's Exokernel.
5. **NanoKernels**: Tiny, focuses on hardware abstraction. Used in embedded systems.

---

### XNU vs GNU Linux

Components of a microkernel-based system include:

1. **Mach Microkernel (Ring 0)**: Manages core services like process, memory, and IPC with message passing.
2. **BSD Component (Ring 0)**: Provides UNIX services, using Mach for file system, POSIX API, and networking.
3. **System Call Interface**: Connects user apps to kernel services, converting POSIX calls to Mach messages.
4. **I/O Kit (User Space)**: A C++ framework for device drivers, using object-oriented principles for safety.
5. **Virtual Memory System**: Uses Mach VM for memory management, featuring protection and demand paging.
6. **Interprocess Communication (IPC)**: Enables message passing for data exchange and synchronization.
7. **Scheduler**: Employs Mach's scheduling for task management and CPU resource allocation.

---

# Much like CTFs, a hacker's goal is switching
# from user land to kernel land, either 
# directly or indirectly.

--- 

### A normal boot process for a kernel

&rarr; The BIOS is not loaded, it stored on a chip in the motherboard <br>
&rarr; When you turn on a computer, it goes through 
POST - power on self test. This generally checks for bootable devices<br>
(floppy, CD, USB, HDD, etc.).  <br>
&rarr; The BIOS then checks for bootable devies for a boot signature that indicates the device is bootable. <br>
&rarr; If a bootable device is found, the BIOS transfers control to the boot loader, initiating the OS startup process. <br>
&rarr; If no bootable device is found, an error message is displayed, prompting for system check or setup. <br>
&rarr; Once the OS starts, the kernel initializes, setting up memory management, device drivers, <br>and system call interfaces for user space applications. <br>
&rarr; The kernel then mounts the root filesystem and starts init (or an equivalent init system) <br>to begin the user space initialization process. <br>
&rarr; Kernel modules can be loaded or unloaded dynamically, providing flexibility in managing hardware drivers <br>and system features. <br>
&rarr; Interrupt Service Routines (ISRs) are part of the kernel, handling hardware interrupts and<br> ensuring responsive system behavior. <br>

---

### How to switch between rings

#### Defining a Syscall



---



### References

1. https://pwn.college/system-security/kernel-security
2. https://lwn.net/Articles/604287/
3. 