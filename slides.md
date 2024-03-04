---
theme: geist
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
defaults:
  foo: true
transition: slide-left
title: Welcome to Slidev
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

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

### Who is August Radjoe?
&rarr; Intern @ RISC-V, Multiparty.org<br>
&rarr; MS CompSci @ BU <br>
&rarr; Dev @ a few places <br>
&rarr; Open Source @ a few repos, eg. Zephyr
--------------------

# Kernel vs. OS
| Feature        | Kernel           | Operating System  |
| -------------- |:----------------:| :----------------:|
| Definition     | Core component of OS that manages system resources and hardware. | Complete package that includes kernel, UI, and utilities for managing computer resources. |
| Functionality  | Handles low-level tasks like memory management, task scheduling, and device control. | Provides a user interface and a set of applications along with the kernel functionalities. |
| User Interaction| No direct interaction; works in the background. | Direct interaction through graphical or command-line interface. |
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
1. **Linux/UNIX-like kernels**: User space (Ring 3) and kernel space (Ring 0) separation with capabilities for finer access control.
2. **Windows NT-based kernels**: Operates in Ring 0 for the kernel, Ring 3 for user applications, with User Account Control (UAC) for privilege management.
3. **macOS/XNU kernel**: Hybrid approach combining Mach (a microkernel) for message passing and BSD for Unix-like system calls.
4. **Microkernels (e.g., Minix 3, L4)**: Minimal kernel functionality in privileged mode, with services running in user space.
5. **Exokernels**: Direct application access to hardware with management of resources delegated to user-level libraries.

---

### Types of Kernels

&rarr; Monolithic - one giant monorepo that handles all OS-level tasks


(analogy taken from pwn.college)

---

### XNU vs GNU Linux

- **Mach microkernel (Ring 0)**: Handles essential system services like process management, memory management, and IPC, using message passing for communication between tasks.
- **BSD component (Ring 0)**: Adds traditional UNIX services, including the file system, POSIX API, and network stack, leveraging direct calls to Mach for system operations.
- **System Call Interface**: Bridges user applications and kernel services, translating POSIX calls into Mach messages for kernel-level execution.
- **I/O Kit (User space)**: Implements a C++ based framework for developing device drivers outside of Ring 0, utilizing object-oriented principles for reusability and safety.
- **Security Framework**: Integrates access control and authentication mechanisms, such as sandboxing and entitlements, to limit the privileges of processes and drivers.
- **Virtual Memory System**: Manages memory through Mach's VM capabilities, offering features like memory protection and demand paging to ensure process isolation and efficient memory use.
- **Interprocess Communication (IPC)**: Facilitates message passing between processes, enabling data exchange and synchronization across different security domains within the system.
- **Scheduler**: Utilizes Mach's advanced scheduling algorithms to manage process execution, prioritizing tasks and managing CPU resources efficiently.
- **Hardware Abstraction Layer**: Minimizes direct hardware interaction, allowing the kernel and drivers to operate independently of the underlying physical hardware specifics.

---

# Much like CTFs, a hacker's goal is switching
# from user land to kernel land, either 
# directly or indirectly.

--- 

### A normal boot process for a kernel

&rarr; A kernel boots in ring 0, as a kernel