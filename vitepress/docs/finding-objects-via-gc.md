# From a process ID to Mono object addresses

*How to locate the Boehm GC's state in a running Mono process, and generate object addresses straight from its own bookkeeping - without scanning memory.*

::: tip Where this fits in
The [Reading objects](/docs/reading-objects.html#how-to-find-objects) page finds `Window` objects by scanning every mapped memory range for a matching `domain_vtables` value. That works, but it is `O(n)` over the whole process memory. This page is a different, deterministic route to the same kind of result: it walks from the process ID down to the garbage collector's own state, and then generates candidate object addresses by arithmetic instead of scanning for them. It does not (yet) solve reading static fields, but it does answer the "there should be a faster way" note on that page.
:::

---

## Key points

1. **Nothing is scanned or guessed.** Every step follows a pointer that the operating system, the dynamic linker or the garbage collector already has placed in a well-defined structure. The way from "a process ID" to "the garbage collector's state" is deterministic, so no guessing is needed.
2. **One exact address is enough to defeat ASLR.** The kernel tells the process where its own program headers actually are. If we compare that with where they were planned to be, we get the offset that everything was shifted with.
3. **The dynamic linker keeps a kind of public ledger.** It writes down every loaded library, with its load address and its own dynamic section. Once we have found this ledger, we can look up any library without touching files or memory maps at all.
4. **We ask the library itself for its symbol table.** Symbols are found in the same way that the dynamic linker itself finds them, so the result is exactly what the process would resolve, not just an approximation of it.
5. **`GC_arrays` is the anchor point.** Part A ends with one address and one size, and from there the collector's whole bookkeeping can be reached.
6. **Object addresses are generated, not searched for.** The collector already knows which blocks that make up the heap, and how large the objects in each block are. So every object slot follows just by arithmetic: `block start + i × object size`. No memory is scanned for things that look like objects.
7. **The lookup matrix turns the collector's index into a local table.** The two-level block index is read one single time and flattened into a dictionary, so to identify a block does not cost any more reads from the target after that.
8. **The result is a list of slots, not of live objects.** The collector knows where objects *can* be, not what is actually inside a slot right now. Because of this, a short structural check of the first two pointers of each slot is done, to separate real objects from free or stale slots.
9. **Read-only the whole time.** The target process is never stopped, attached to, or modified in any way.

> **About the "Look it up" parts.** Each section ends with a small *Look it up* block, with the commands that open the Linux manual page (or another reference) for the structure that was discussed. If a page is missing on your system, you can install the man-pages package (on Debian/Ubuntu it is `manpages-dev`). The collector's own structures do not have a man page, so for those, the blocks link instead to the Boehm GC source files where they are defined (the Unity fork of bdwgc, pinned to a fixed commit).

---

## 1. The problem

We want to know where the objects of a running Mono program actually live. From outside the process we don't know very much: only a process ID and permission to read its memory.

Address space layout randomization (ASLR) makes this hard, because every time the program starts, the executable and each library get loaded at different addresses. So we do not know where the Mono runtime is, and we also do not know where the garbage collector keeps its state.

**Look it up**
```bash
$ man 5 proc             # search for "randomize_va_space" (the ASLR setting)
$ man 2 personality      # ADDR_NO_RANDOMIZE: how ASLR can be turned off per process
```

## 2. The idea

Instead of searching through memory for things that look like objects, we ask the garbage collector itself. Mono uses the Boehm garbage collector, which needs to know every block of its heap and the size of the objects inside each block. All of this is kept in one central state structure, called `GC_arrays`.

The work is split into two parts:

- **Part A: find `GC_arrays`** (Stages 1 to 4).
- **Part B: turn the collector's tables into object addresses** (Stages 5 to 8).

Every stage gives exactly the one piece of information that the next stage needs:

| Stage | Starts from | Question | Gives |
|---|---|---|---|
| 1 | the process ID, via the kernel's auxiliary vector | Where are the executable's program headers? | load bias and the address of the executable's dynamic section |
| 2 | the executable's dynamic section | Where is the dynamic linker's ledger? | address of the ledger (`r_debug`) |
| 3 | the ledger | Which libraries are loaded, and which one is the Mono runtime? | load base and dynamic section of the Mono runtime |
| 4 | the Mono runtime's symbol tables | Where is the symbol `GC_arrays`? | **`GC_arrays`: address and size** *(end of Part A)* |
| 5 | `GC_arrays` | Which memory ranges belong to the heap? | the list of heap sections |
| 6 | `GC_arrays` (its block index) | What is inside each 4 KiB block? | the lookup matrix (block → header) |
| 7 | heap sections + lookup matrix | How big are the objects in a block, and where do they start? | candidate slots (address, size) |
| 8 | the candidate slots | Does the slot really hold an object? | **confirmed objects (address, size)** *(end of Part B)* |

---

## 3. Stage 1: the kernel hands over the first address

When the kernel starts a program, it hands over some facts to it, in a small table called the **auxiliary vector**. From outside it is visible as `/proc/<pid>/auxv`.

Two of its entries are what we actually need:

- the runtime address of the executable's **program headers**, and
- the number of program headers.

Program headers are the executable's own description of its segments: what is loaded where, and what the dynamic section is. In the file they exist as they were planned at link time already, with planned addresses.

The kernel gives us the address where the headers *really* are. One of the headers describes the header table itself, and it holds the address where it *should* have been. The difference between these two is the **load bias**, which is the amount that ASLR shifted the executable with:

```
load bias = real address of program headers − planned address of program headers
```

From here, every planned address in the executable can be turned into a real one, just by adding the load bias. Nothing had to be searched for, to get this far.

**Look it up**
```bash
$ man 3 getauxval        # the auxiliary vector and its AT_* entries (AT_PHDR, AT_PHNUM)
$ man 5 proc              # search for "auxv": the /proc/<pid>/auxv file
$ man 5 elf                # program headers, and the PT_PHDR / PT_DYNAMIC segment types
$ man 8 ld.so             # search for LD_SHOW_AUXV

# to see it live
$ LD_SHOW_AUXV=1 /bin/true      # prints the auxiliary vector of a starting program
$ readelf -l /bin/true            # prints the program headers
```

## 4. Stage 2: the dynamic section and the linker's ledger

One of the program headers points to the **dynamic section**, which is a list of tag/value pairs describing how the executable is linked: which libraries it needs, where its symbol tables are, and more. With the planned address plus the load bias, we can find it in memory.

One entry in this list is a bit special. In the file on disk it is only an empty placeholder, called the **debug entry**. But when the program starts, the **dynamic linker** fills it with the address of its own public state block, the ledger `r_debug`. It exists so that debuggers can find out which libraries are loaded, and we use it for exactly the same purpose.

By reading this one entry, we get the dynamic linker's ledger.

**Look it up**
```bash
$ man 5 elf              # search for "Dynamic section" and DT_DEBUG
$ man 8 ld.so             # the dynamic linker itself

# r_debug does not have a man page, its definition is in the C header
$ less /usr/include/link.h      # search for "struct r_debug"

# see the dynamic section of any executable
$ readelf -d /bin/ls
```

## 5. Stage 3: the list of loaded libraries

The ledger contains the head of a **linked list, with one entry for every loaded object**: the executable itself, the kernel-provided helper library, and every shared library. Each entry tells us three things that matter to us:

- the library's **load base**, which is the address it was actually mapped at,
- the **path** it was loaded from, and
- the address of the library's own **dynamic section**.

This is the big advantage of using the linker's ledger: we do not need to parse `/proc/<pid>/maps` or read library files from disk. The linker has already done the bookkeeping, and it is always consistent with what is really loaded.

**Look it up**
```bash
$ man 3 dlinfo            # search for RTLD_DI_LINKMAP: shows the link_map structure
$ man 3 dl_iterate_phdr   # the supported way to list loaded objects from inside a process
$ man 7 vdso               # the kernel-provided helper library that also appears in the list

$ less /usr/include/link.h      # struct link_map and struct r_debug
```

### Choosing the Mono runtime
The list is walked entry by entry, and the Mono runtime is picked out **by the file name** of each library: it must contain `mono`, and it must not look like one of the other Mono-related libraries that live next to it (debug, SGen, native support and similar ones). Only the file name is looked at, not the directories above it, so an unusual install path can not lead to a wrong choice.

The walk itself is exact. It is only the choice of *which* entry is the Mono runtime that is a name-based decision. This is done on purpose, as a simple rule that works for a target we already know.

What we keep from the chosen entry is its load base and the address of its dynamic section.

## 6. Stage 4: looking up `GC_arrays`

The Mono runtime's dynamic section describes the library's symbols. Three things are needed from it:

- the **symbol table**, which lists every exported symbol together with its location and size,
- the **string table**, which holds the symbol names, and
- the **GNU hash table**, which is an index that makes it fast to find a symbol by name.

The dynamic linker has already adjusted these entries to real runtime addresses when it loaded the library, so they can be used directly, without any extra work.

### How a symbol is found by name
The GNU hash table works a bit like the index of a book. It is built so that a name can be found without comparing it against every single symbol:

1. **Hash the name.** A simple, fixed function turns the name into a 32-bit number.
2. **Quick rejection.** A small filter in front of the table can already say "this name is definitely not here". Most names that don't exist stop right there.
3. **Pick the bucket.** The hash chooses one of a fixed number of buckets. A bucket points to the start of a short **chain** of symbols whose hashes fall into that bucket.
4. **Walk the chain.** Every chain entry carries the hash of its own symbol. Where the hash matches, the real name is read from the string table and compared with the wanted one. A marker on the last entry of a chain tells where to stop.

This is the very same procedure that the dynamic linker runs every time a program calls a library function. We just repeat it by hand, from outside the process.

### Turning the entry into an address
The symbol table entry stores the symbol's location as an **offset from the start of the library**, not as an absolute address. So the last step is the only place where the load base is actually needed:

```
address of GC_arrays = load base of the Mono runtime + symbol offset
```

The entry also stores the symbol's **size**, which we keep as well.

**Look it up**
```bash
$ man 3 dlsym             # the C function that does this lookup by name inside a process
$ man 3 dladdr             # the reverse direction: from an address to the symbol containing it
$ man 3 dlopen             # how a library gets loaded and handed to dlsym
$ man 5 elf                 # symbol table entries (Elf64_Sym: value, size), string tables
$ man 1 readelf            # the tool used below

# The GNU hash table has no man page of its own, so inspect it with readelf instead
$ readelf -d <libmono.so> | grep GNU_HASH      # its address in the dynamic section
$ readelf -I <libmono.so>                        # bucket statistics of the hash tables

# See a symbol's value and size in the same way this document resolves it
$ readelf --dyn-syms <libmono.so> | grep GC_arrays
$ nm -D <libmono.so> | grep GC_arrays
```

---

## 7. First result: `GC_arrays`

The outcome of Part A is two numbers:

- **the address of `GC_arrays`** in the target process, and
- **its size** in bytes, as declared by the library.

`GC_arrays` is the garbage collector's central state. Among other things, it contains the list of memory ranges that the heap consists of, the entry to the collector's block index, and a sentinel value that the index uses to mean "empty". This is where Part B starts.

---

## 8. Stage 5: the heap sections

At a fixed position inside `GC_arrays`, the collector keeps the list of its **heap sections**: every range of memory it has requested from the operating system for its heap, stored as (start address, size). The list ends at the first entry whose start address is 0.

This is the outer boundary of the search space. Everything the collector manages lies inside these ranges, and nothing outside them does. What the list does not tell us is what is actually inside: a section is just raw memory that the collector has divided into blocks.

Where exactly the list sits inside `GC_arrays` depends on how the collector was compiled. It is a fixed offset, which is already known for the reference build of the target (see Section 13).

**Look it up**

The collector's structures do not have a man page. They are defined in the Boehm GC source instead. The links point to the Unity fork of bdwgc, at a fixed commit.

**[`include/private/gc_priv.h`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h)** - the struct definitions:
- [`struct _GC_arrays`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h#L1243), with the fields [`_heap_sects`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h#L1394), [`_top_index`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h#L1416) and [`_all_nils`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h#L1310)
- the block header, [`struct hblkhdr`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h#L1020) (called `hdr` in the code)
- [`bottom_index`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_hdrs.h#L119)

**[`headers.c`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/headers.c)** - how an address is mapped to its block header: top index, bottom index, forwarding values.

**[`doc/`](https://github.com/Unity-Technologies/bdwgc/tree/1113fefc4ba0e895767ac780d84e9e456fd624db/doc)** - the collector's own documentation, including the algorithmic overview (`gcdescr`).

## 9. Stage 6: the block index and the lookup matrix

### Blocks and block headers
The collector divides its heap into **blocks of 4096 bytes**. A block only serves one single object size and one object kind: a block for 48-byte objects holds only 48-byte objects, nothing else. Every block has a **block header**, which is kept somewhere else and describes it. Five of its fields matter for us here:

- the block's **own address**,
- the object **size**, meaning the size of each slot in the block,
- the object **kind** (for example pointer-free data compared to objects that contain pointers),
- some **flags**, and
- a **descriptor**, which is non-zero for a block that is actually in use for objects.

### How the collector finds a header
Given any address, the collector has to find the header of its block fast. For this it uses a two-level table, the same idea as a CPU's page tables. The address is first turned into a **page number** (`address >> 12`), which is then split into two parts:

| Part of the page number | Bits | Selects | The slot holds |
|---|---|---|---|
| upper part | the next 11 bits | a slot in the **top index** (2048 slots) | a pointer to a bottom index |
| lower part | the lowest 10 bits | a slot in that **bottom index** (1024 slots) | a pointer to the block header |

Not every entry is actually a pointer to something real:

- A top slot that holds the **`all_nils` sentinel** (kept in `GC_arrays`) or 0 means "nothing here". The sentinel points to one shared, empty bottom index.
- A bottom slot that holds a **small number** (at most one block size) is not a pointer at all. It is a "forwarding" value, used for the follow-up blocks of an object that spans several blocks, so it is just skipped.
- Every other bottom slot is a pointer to a block header.

### Building the lookup matrix
To resolve each block with three reads from the target would be slow, because the index can address 2048 × 1024 = 2,097,152 blocks (8 GiB of heap), even though only a small fraction of it is actually populated. Because of this, the index is read **once, completely**:

1. read the whole top index,
2. for every top slot that is not empty, read the whole bottom index,
3. for every real header pointer, read the header fields that were listed above.

The result is stored in a local dictionary:

```
key = (top slot << 10) | bottom slot   ──►   header fields of that block
```

This is the **lookup matrix**: a table over two coordinates (top slot, bottom slot), flattened into one single key. Once it exists, any block can be identified just by arithmetic (compute its two slot numbers, combine them into the key) and one local dictionary lookup. The target does not have to be touched again after that.

**Look it up**
```bash
$ man 2 process_vm_readv   # reading many ranges of another process in one call (bulk reads)
$ man 2 pread                 # reading one range at an explicit offset
```

The index and header definitions are in the same two files as Section 8: [`headers.c`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/headers.c) and [`include/private/gc_priv.h`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/include/private/gc_priv.h).

## 10. Stage 7: generating the candidate slots

The heap sections give us the ranges, and the lookup matrix gives us the meaning of each block. Together, this is already enough to generate object addresses.

For every heap section, we step through it one block at a time (4096 bytes). For each block:

1. **Look it up.** Compute the block's key and look it up in the matrix. If there is no entry, the block has no header, so it is just skipped.
2. **Filter.** Skip the block if its size or descriptor is zero (it is then not in use for objects), if its object kind is not the kind we are searching for, or if it carries the flag for special large allocations.
3. **Generate the slots.** All objects in a block have the same size, and the first one starts right at the beginning of the block. So slot *i* lies at

```
slot address = block start + i × object size        for i = 0 … (4096 ÷ object size, rounded down) − 1
```

For example, with 48-byte objects a block has 85 slots, and the last 16 bytes are left unused:

| | slot 0 | slot 1 | … | slot 84 | unused |
|---|---|---|---|---|---|
| **Offset in the block** | 0 | 48 | … | 4032 | 4080 |
| **Size** | 48 B | 48 B | … | 48 B | 16 B |

Every address produced here follows just from the collector's own record of the block. Not a single byte of an actual object has been read at this point.

### Restricting to one object kind
The heap contains blocks of several different object kinds. If we only generate slots for the kind that holds the objects we search for, the candidate set becomes much smaller. On the reference build this brings roughly 45,000 candidates down to roughly 5,000, while it still contains every object the later stages need.

### Objects larger than a block
An object that is larger than 4096 bytes spans several blocks, and only its first block has a real header. Because its size is larger than the block, such a block gives zero slots, so these objects are not generated at all. This concept only covers objects that fit into one block.

### The output
The result of this stage is a list of `(address, size)` pairs, where the size is the slot size of the block.

## 11. Stage 8: from slots to objects

### What the collector knows, and what it does not
The collector's bookkeeping tells us **where objects can be** and **how large their slots are**. It does not tell us what a slot actually holds right now, because the collector is only a general memory manager and knows nothing about Mono's object layout. A slot can be in one of several states:

| State of the slot | Its first word | Passes the check below? |
|---|---|---|
| Live object | pointer to the vtable (runtime metadata, outside the heap) | yes |
| Dead object, not yet collected | pointer to the vtable | yes, it is still structurally intact |
| Free slot | link to the next free slot, which lies inside the heap | may pass, because free slots point to each other |
| Never used, or stale data | 0 or leftover bytes | usually not |

So the list from Stage 7 is really a list of **slots**, and it is the last stage that separates the real objects from the rest.

### The two-pointer check
Every Mono object begins with a pointer to its **vtable**, and the first field of a vtable is itself a pointer to its **class**. For each candidate slot:

1. Read the first word of the slot. This is the **vtable address**, and it must be readable.
2. Read the first word at the vtable address. This is the **class address**, and it must also be readable.
3. The slot address, the vtable address and the class address must be **pairwise different** from each other.

> **Proposed refinement (still to be verified on the reference build).** The vtable and the class are runtime metadata that Mono allocates outside the managed heap, so in a real object, neither address lies inside a heap section. A free slot, on the other hand, points to another slot that is *inside* the heap. If we test the two addresses against the heap-section list from Stage 5, the two cases can be separated without any extra reads at all. This can be confirmed by counting, over all objects that were mapped successfully, how many vtable or class addresses fall inside a heap section; the expected number here is zero.

### Why this does not go against "generate, don't verify"
The addresses are still **generated** from the collector's bookkeeping, and no memory is scanned in order to find objects. The check only decides, for the slots that were already generated, whether each one really holds an object or not. A scan-and-verify approach would have had to run this check on every single address in memory, but here it only runs on a few thousand candidates, which is why such a short check is enough.

### What the check cannot decide: alive or dead
A dead object that the collector has not swept yet still has an intact vtable and class, so it passes every structural check too. Whether an object is still in use or not cannot be read from the slot itself. It is better to establish this by **following references from an object that is already known to be live**, rather than just taking the first match of a type.

## 12. The result: object infos

The final result of Part B is a list of `(address, size)` pairs, for the slots that passed the check. For each of them the vtable and class addresses are already known, so the objects can be grouped by type: all objects with the same vtable belong to the same class.

From here, [Reading objects](/docs/reading-objects.html) takes over: the class addresses found here match the `domain_vtables` values used on that page, so the candidate slots from this page can be used directly, instead of scanning the process memory for them.

---

## 13. What this concept relies on

### Assumptions
- **A 64-bit Linux target**, with the standard dynamic linker, whose ledger, library list and in-place adjusted dynamic sections are used exactly as described.
- **The Mono runtime exports `GC_arrays`** in its symbol table. If the symbol is not exported, this whole route does not work, and another way to locate the structure would be needed.
- **The Mono runtime can be identified by its file name**, among the loaded libraries.
- **One known build of the collector.** The positions of the heap-section list, the top index and the `all_nils` sentinel inside `GC_arrays`, together with the geometry of the index (2048 top slots, 1024 bottom slots, 4096-byte blocks), are fixed values that only hold for the reference build.
- **Mono's standard object layout.** Every object starts with a vtable pointer, and every vtable starts with a class pointer.
- **Permission to read the target's memory**, through `/proc`.

### Known limits
- **Slots, not live objects.** See Stage 8.
- **Objects larger than one block** are not generated at all.
- **The object-kind restriction is tuned to the reference build.** This is what makes the candidate set small, and it must be reconsidered for a different build or a different search target.
- **The lookup matrix key does not contain any high address bits.** Each bottom index of the collector carries a `key` with the high address bits of the region it covers, but the matrix key `(top slot << 10) | bottom slot` does not have this. On 64-bit builds the collector hashes its top index, so two heap regions that are far apart from each other can still end up in the same top slot. The block's own address, which is stored in its header, can be compared with the address of the block being looked up, to detect such a collision.
  → how the reference build resolves this is visible in [`headers.c`](https://github.com/Unity-Technologies/bdwgc/blob/1113fefc4ba0e895767ac780d84e9e456fd624db/headers.c).

**Look it up**
```bash
$ man 5 proc              # search for "/proc/[pid]/mem"
$ man 2 ptrace             # search for "ptrace access mode": who is allowed to read another process
$ man 2 pread                # reading at an explicit offset, without moving a file position

# Check the "exports GC_arrays" assumption for a given library
$ nm -D <libmono.so> | grep GC_arrays
```

---

## 14. Where this leads

With the confirmed object list, every object of the managed heap is already known by address, size, vtable and class. What is left is to name the types (class → namespace and name) and to read their fields, which is exactly what [Reading objects](/docs/reading-objects.html) already covers - this page only replaces how the candidate addresses were found in the first place.
