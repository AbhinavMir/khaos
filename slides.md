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

To I.C.E: I am NOT A HACKER! I am just an international student who likes kernels.

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

### Structure of the talk

1. Basics of a kernel
   1.  Kernels vs OS, Privilege separation, Types of kernels, XNU vs Linux, Boot process, Switching between rings, How a Syscall works (an an example to how to dive into reading linux), How does a kernel module, Why can buffer overflows happen
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

<center><img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/600px-Priv_rings.svg.png" width="350"></center>

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

# And the mitigation tactics
# Are pretty similar to what you would
# do in a defensive CTF team
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
Eg. `read()`


```
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count) // 3 for 3 arguments
{
	struct fd f = fdget_pos(fd); // get file descriptor
	ssize_t ret = -EBADF; // default error code

	if (f.file) { // if file descriptor is valid
		loff_t pos = file_pos_read(f.file); // get file position
		ret = vfs_read(f.file, buf, count, &pos); // read from file - virtual file system, buffer means where to read to, count is how many bytes to read, pos is the position to read from
		if (ret >= 0) // if read is successful
			file_pos_write(f.file, pos); // update file position
		fdput_pos(f); // release file descriptor
	}
	return ret; // return read status
}
```

This function in turn opens up `vfs_read()`, which is a virtual file system read function that fails if we have no read persmission, are neither synch or asynch, if the buffer isn't a valid read area. 

---

### Dissecting the syscall()

<center><img src="https://www.usenix.org/legacy/publications/library/proceedings/usenix01/full_papers/kroeger/kroeger_html/linux.caches.gif" width="400"></center>

`vfs_read` is a part of the virutal file system abstraction that provides uniform access to different file systems. It is a part of the kernel that manages file I/O operations, including reading and writing data to and from files. Eventually, it leads to reading from the disk using assembly at some point. 

---

### Visualizing a Syscall

Depending on how you use [KConfig](https://elixir.bootlin.com/linux/v3.14/source/kernel/trace/Kconfig). You can visualize the syscall using `strace` or `ltrace` to see what system calls are being made. 


---

### Revisting Spectre variant 2

- Variant 2 (Branch Target Injection or CVE-2017-5715): This variant involves poisoning the branch predictor, a component of the CPU that guesses which way a branch (e.g., an if statement) will go before it is known for sure. By manipulating the predictor, an attacker can make the CPU speculatively execute instructions at an attacker-controlled address, potentially leaking sensitive data through side effects similar to Variant 1.

- Mitigation: Solutions include processor microcode updates to control speculative execution more tightly (IBRS - Indirect Branch Restricted Speculation) and software-based mitigations like "retpoline" that avoid speculative execution through indirect branch predictions. (Let's talk about this now)
- Problems with the mitigation strategy

First, they add overheads. 

It slows down the system by preventing indirect branch predictions from being shared between threads on the same core in hyperthreaded (SMT) processors. This isolation means that each thread must wait for actual branch directions to be resolved rather than benefiting from predictions made based on past executions. This lack of prediction increases the number of CPU cycles needed to complete tasks, reducing overall performance.

---

"Yes, Intel calls it "STIBP" and tries to make it out to be about the indirect branch predictor being per-SMT thread.
But the reason it is unacceptable is apparently because in reality it just disables indirect branch prediction entirely. So yes, *technically* it's true that that limits indirect branch prediction to just a single SMT core, but in reality it is just a "go really slow" mode."

<center><img src="https://banner2.cleanpng.com/20180526/gba/kisspng-linus-torvalds-linux-kernel-gnu-linux-history-of-l-5b09d3f2ae4578.3916557915273707387138.jpg" width="400">
A polite Linus</center>


---

### Did the mitigations work?

There is a fairly recent exploit (on a kernel newer than mine) - specifically on one (unnamed) major cloud provider. But the scope is massive enough for us that rewriting a mainline stable kernel version to tackle this in a performant efficient way isn't feasbile. A lot of people still use STIPB.

- The attack involves two processes: one acting as the attacker and the other as the victim.
- The attacker process speculatively poisons an indirect call to redirect it to a target address.
- The victim process tries to mitigate the attack by either calling `PRCTL` or writing to the MSR directly.
- Despite attempts to enable Spectre v2 mitigations (such as IBRS or STIBP), a bug in the Linux Kernel (prior to version 6.3) leaves userspace threads vulnerable to cross-thread branch target injection on systems with plain IBRS enabled.
- The vulnerability stems from an optimization that disables STIBP if mitigation is set to IBRS or eIBRS, which does not protect against SMT attacks on userspace as effectively as eIBRS does.

---

### References

1. https://pwn.college/system-security/kernel-security
2. https://lwn.net/Articles/604287/
3. https://lwn.net/Articles/604287/