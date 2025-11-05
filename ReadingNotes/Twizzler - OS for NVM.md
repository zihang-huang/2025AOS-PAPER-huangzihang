[Twizzler: a Data-Centric OS for Non-Volatile Memory](https://www.usenix.org/conference/atc20/presentation/bittman)

The two-tier memory hierarchy split between high-latency persistent storage and low lantecy volatile memory may evolve into a single level of large, low latency and direct-accessable persistent memory.(the NVM)

The direct access nature of NVM invites the use of load and store instructions to directly access persistent data.
- Enabling persistent data manipulation.
- Wiithout the need to transform data between in-memory and on-storage data formats.

# Traditional OS
Traditional OS are process-centric.

Voltatile memory(RAM) and persistent storage(disk, SSD, or NVM used as block storage) are fundamentally separated.
- Volatile memory holds live process data(heap, stack, cache) that disappears on reboot.
- Persistent storage: holds files, databases, or other on-disk data structures that survive power loss.
CPU cannot directly access files on disk, apps must serialize data before persistence and deserialize it when reading back into memory.

Serialization means converting complex, in-memory data structures(with pointers, references, etc.) into a flat, linear format that can be written to disk or sent over a network.
- RAM pointers are valid only in the process's virtual address space, and those addresses have no meaning after reboot or across different processes.
- Usually, app decides to persist data, like the struct or trees that need to be stored.
	- And then the in-memory representation is converted to bytes in JSON or other custom routines.
	- System call to write data, to pass data to the kernel
	- Kernel then copies data into the page cache, marks it dirty for later flush to disk
	- Kernel or FS eventually flushes the data to persistent storage

Current OS also implements virtual memory address, it’s **not** a physical hardware address — it’s an **abstraction** managed by the operating system and the CPU’s **Memory Management Unit (MMU)**.

![[Pasted image 20251014223106.png]]

Traditional OS utilizes `mmap()`(memory map), which is a syscall, to lets a process map files or devices into its virtual address space.

![[Pasted image 20251014223236.png]]


## The Data-Centric OS
NVM characteristics:
- Low latency
	- The cost of a system call to access NVM dominates the latency of the access it self.
- The processor can directly access persistent storage using load and store instructions.

Direct low-latency access to memory means explicit searlization is nonsense for NVM.
- Serialization adds extra cost for r/w data in different formats, and the transformation between them.
- We should design the programming model around *in-memory* persistent data structures.

**The basic requirement for OS to most effectively use NVM:**
1. Remove the kernel from the persistent path
	- Because Syscalls to persist data are costly.
2. Design for pointers that last forever
	-  The pointer should have same lifetime as the data they point to.

`mmap` is not suitable for NVM structure, first it still has significant kernel involvements. Secondly, operating on persistent data either require programmer to use fixed virtual addresses, which cause lots of other problems, or use virtual address but still has ephemeral problems etc.

Even NVM-aware libraries like **PMDK** retrofit persistence into existing POSIX abstractions, leading to:
- **Complex coordination** (e.g., shared object IDs across machines)
- **Limited scalability**
- **No long-lived or cross-context pointers**
Single-address-space OSes (e.g., Opal) address only part of the problem—they remove some process boundaries but still depend on ephemeral virtual addresses.

## A Data-Centric Approach - Persistent Pointers
Persistent pointers encode a persistent identification of data instead of an ephemeral address.
- Any thread can access the desired word of memory regardless of address space.
- These are valid system-wide and across time.

Processes are non-necessary, instead OS-wide security context can used to define access rights and views to define how data objects are mapped into address spaces.

# The Design of Twizzler
Twizzler is implemented as a **standalone kernel** plus a **userspace runtime** called `libtwz`, which applications link against (alongside a modified `musl` libc). A compatibility library, **twix**, emulates Linux syscalls for legacy code.

![[Pasted image 20251014225756.png]]

### 3.1 Object Management
- All persistent data is organized into **objects** identified by unique 128-bit IDs.
- Objects are contiguous memory regions (from 4 KiB – 1 GiB).
- The **object** is the **unit of protection**, not the process.
- Objects can be created, deleted, or copied (using COW semantics).
- Reference counts determine lifetime—once unmapped everywhere, an object may be deleted.

### 3.2 Address Space Management: Views
- A **view** is a persistent object that defines an address space layout (a mapping of object IDs to virtual addresses and access bits).
- When a page fault occurs, the kernel consults the view to determine what object should be mapped.
- This mechanism allows **userspace to manage mappings** dynamically without system calls.
- Two system calls support this:
    - `set_view` — switch to a different address space layout.
    - `invalidate_view` — notify the kernel of mapping changes.

Views enable lightweight, flexible mappings and isolation between threads or programs.


### 3.3 Persistent Pointers and the FOT
Twizzler introduces **cross-object pointers**, which store a **(FOT index : offset)** pair instead of a virtual address.
- Each object contains a **Foreign Object Table (FOT)** listing the external objects it references.
- The FOT provides **indirection**: pointers don’t store full 128-bit object IDs, keeping them 64 bits long.
- The indirection allows:
    - **Late binding** (e.g., replace a library or data object transparently)
    - **Access control per pointer**
    - **Machine-independent, sharable data**
Pointer translation occurs through `ptr_lea()` (to load effective address) and `ptr_store()` (to store persistent form). These are lightweight and cache-friendly—typically < 0.5 ns added latency.

### 3.4 Security and Access Control
- Security is **hardware-enforced** by MMU permissions.
- Twizzler uses **late-binding access control**: a thread can request broader permissions than needed, but only actual violations (e.g., unauthorized writes) trigger faults.
- **Security contexts** replace the Unix “process user/group” model; they persist across executions and specify object-level permissions.
This model enables **fine-grained, persistent, and flexible** security without adding kernel overhead.

### 3.5 Crash Consistency
Twizzler provides:
- **Low-level primitives** for flushing cache lines and enforcing ordering (fences).
- A **transactional logging mechanism** (`TXSTART–TXEND–TXRECORD`) for durable updates.
- A **resume mechanism**: after a power loss, threads restart at a `_resume` point with their previous view remapped, enabling recovery without full reloads.

### 3.6 Implementation Notes
- The kernel is **tiny**—about 11 K lines of code total.
- It’s architecturally similar to an **Exokernel**: most complexity lives in userspace.
- An initial prototype was built on FreeBSD 11 to test ideas before implementing a standalone kernel.