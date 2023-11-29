[[OsHW]] | [website](https://www.cs.columbia.edu/~nieh/teaching/w4118/) |  [Ed](https://edstem.org/us/courses/45654/discussion/) | [Study Guide](obsidian://open?vault=MaraNotes&file=Y3S1%2FMidterms%2FOS%20Study%20Guide)


# Filesystems
November 28

--- 
* File: Abstraction of persistent storage (but UNIX said everything is a file)
* Filesystem: The way that we organize a chunk of bytes into files so we can do all the file operations and also organize them
* Other way of thinking about thing, file is a comonly used API, filesystem is implementation of API
## ramfs 
* `fs_initcall(function)` initializes the file system 
* register_filesystem
	* registers a type, which involves a name functions, and flags
* There are generic filesystem operationsthat we want to look at and use
	* like block_read_full_page(), block_read_full_folio() etc.
* Operations:
	* name
	* init_fs_context
		* Modern version of mount
		* tells you what internal operations you need to do to do certain things
		* under the covers mount_bdev()
		* What you want to do is read the medatata from the filesystem's superblock, and populate the in memory represenation of the superblock
		* We call get_tree
	* parameters
	* kill_sb
	* flags
* Metadata is stored in the superblock

## Homework 6
* Create a filesystem for a disk
	* (Assume pages and disk blocks are 1 to 1)
	* Because disks are a whole ordeal we're gonna take in a file and treat is like a disk

### Part 1
#### Important Commands
* Format and Mount a disk
* **dd** command creates a 100k file with a bunch of 0s
*  **mkfs** -t ext2 /dev/loop0
	* make the file at /dev/loop0 an ext2 filesystem 
* **mount \<FILE> \<FILE>**
	* first file is the file system, second file is the directory that we're attaching the filesystem into
* Output from **ls -al**
	* total \<total>
	* \<write perms> \<number of links> \<file owner> \<file group> \<Size> \<timestamp> \<name>
	* links include self and parrent
* **unmount** remove file system
* **detatch** turns it back into file
* **mkefs** - We need to make an implement one that lets this work
* **insmod** load kernel module (precompiled code)
#### Assignment
Implement a new filesystem as a kernel module
* ./format_disk_as_ezfs \<index> \<size>
* write the bits into the file such that if you mkfs & mount the system it interprets it as 
* compare to ez-ARCH



2 Part assignment



# Page Replacement, Page Performance, and filesystems Nov 21

## Better Swapping algorithm
* To Be clear, Swapping is when we move somethign from memory into physical memory. When we need to access memory and memory is full, put a frame of memory on disk
* If we do fifo swapping the more frames we get, the more faults we have
	* If we could look intot he future, then we could just pick the one that's least likely to be picked in the future
	* We can't, so we do **Least Recently Used (LRU)**: If something hasn't been used recently it probably wonn't be used int he future, so swap out the leeast recently used frame, the problem is we don't have a timestamp, so we can't really implement this
	* **1 Bit Access Bit**
		* each PTE also has an access bit
		* When it's accessed, set the bit to 1
		* then the swapper will do a test and set, setting every accfess bit to 0, if we find one that's already 0, then that's what we consider the "least recently used" and we swap that
	* Some systems will not do this though, because of all the memory we have on modern systems
## Alternative to swap on full
* 2 types of memory: Anonymous memory, and file backed memory
	* On full, just push file-backed memory back onto disk where they were before
		* Or if there isn't any change on memory so we can just free the pages

## Paging performance
* If every instruction causes a page fault things will slowww dowwwwwn they will slowww ay down
* This is called **trashing**
	* To avoid it linux has an Out of Memory Killer: Kills processes to free memory
* Every process ahs what we calla  "working set": The amount of memory it needs to work
* Page size impacts paging perfomrance: If you have larger pages means less page faults and less translations, 
* Memory Layout: What'st he order in which you access things, how do you access memory, the more efficient it is, the better optimized it is
## Filesystems: Memory 2 
* Beginning assumptions: All the opertaions are in memory, when we're done doing operations, we write it to disk
* **Page Cache**: Where non anonymous memory goes
### Block Devices
* Conceptualize a disk as containing 0 to n blocks of memory
	* There's an on disk representation of these blocks (filesystem)
	* Disk has metadata that defines the datr ablcoks as well as an index node: What file are we on rn
* When we write stuff to disk we first write it to teh cache and later it'll go to the disk
* **Buffer Head Interface**: How we interact with the page cache
	* ```sb_bread```: Allows us to specify some block of data and put it into ram
	* ```mark_buffer_ditty ``` : Once we've modified data, we do this so that the swapper knows you can't just get rid of this memory, you need to write it back to the block device
	* brelse: Means we're done

### Linux Filesystems
* Linux supports many filesystems, similar to supporting many different schedulers
* uses VFS infrastructure: Virtual file system infrastructure
	* Defines common operations a filesystem needs
#### Common Filesystem Operations
* load fs/init
* Mount filesystem: plug in this filesystem into whatever current filesystem we have
* open, read, write, etc.

#### Parts of VFS
* Superblock
	* inode: memory representation of one item)
	* dentry: Directory Entry: link between a path and an item 
	* Process specific Structures
		* FILE: Every open file (FD)
			* In the task struct we have the files_struct which has a list of FILEs. a File descriptor is just the idex of the relevant file struct
	* 


# Finishing Discussion of memory management - Nov 16

**Questions**
* how does static allocation work here?
* how does writing to memory memory in usremode correspond to the acutal physical thing

* when we copy to user, how can we tell if it fails?
	* Copy to user is just a memcpy
	* The page fault handler will return a certian value if it's memory without valid virtual address space, and copy to user will detect that and return a non 0 value
## Page fault handler
* There's a table fault_info that tells you what function to do on a given page fault
* Key function: Do_page_fault (in x86 and arm)
	* Goes to \_\_do_page_fault()
		* First it gets the vma of the current runing process's mm_struct
			* find_vma will either find the given VMA for a given address, or the next allocated VMA
			* It might find the next VMA for the case where the buffer you were calling to is on the stack instead of the heap
			* In this case, expand the vma to include the address for the statically allocated memory
			* this is the growsdown case for the homework: look in fault.c line (490)
	* \_\_do_kernel_fault
		* There are certain situations in which we expect page faults, so if we don't get any, we'll do cleanup, then return someting to tell any callers that used this function that this happened

### For Kernel_Clone
* If itIn the case of a new thread, then we'll just use the same mm struct since we're point ing to the same memory
* Other case: calls dup_mm
	* Allocate an MM, memcpy the fields in the mem struct
		* (doesn't copy the things that they point to)
	* Then call dup_mmap
		* Which will copy all the fields on the inside
		* Which will then call copy_page_range
			* Since most of the page table isn't in use, the move is instead do a  loop through each vma in the page table, and then copy everything only in the range of the vmanv
				* Currently Page table points to the same parts of physical memory 
			* Then it uses a strategy called copy on write. The memory is set as read only, when you reach it, you get a page fault, which then calls do_page_fault, which will eventually go to the handler for copying a write protected copy, which will mark it as writable first, and then copy it, figure out the pfn it would ocrrespond to, and then change the pte to point to the new memory we wrote to, then we do **ptep_clear_flush**, which flushes the cache  
>[!TLB info] The TLB doesn't just contain from the va->pa translation, it also has an address-space-identifier for each entry, which lets it know which address spaces correspond to which translations, so you only need to flush the TLB whenever you're changing the virtual address to physical addres translation
### Page Replacement
* When we're out of pages, we look through the pages we have, and see if there's any that aren't being used, and then frees them
* In code:
	* k**swap**d: scans memory to find pages to free replace
	* It'll be called either if we're out of pages, or if the number of pages is less than some threshold (this case is more common) 
	* we put it on the disk, and then there's a bit on the PTE called the present bit, which tells the pte that the frame that they're pointing to isn't there. so that if the process tiers to reach that, it'll page fault, and the handler will swap it back
* Belady's anomaly, if we're fifo, then the more ram we have, with the swaps case, we don'
# Paging and code - Nov 14
* Precesses 
	* * processes represent memory
	* - ﻿﻿get_task_mm returns the mm associated with a task struct
- ﻿﻿mmput decrements the use count and releases all resources for an mm

## Semaphores
* Two Operations: Down and Up
	* Down is equivalent to lokcing up is unlockign
	* Conceptually decrements nad increments a counter
* The way lock works:
	* if count > 0 
		* Decrement count
		* else
		* Down 
		* unlock

```python
Down()
   if count > 0
   Grab lock and decrement
   otherwise release all locks go sleep and get on a 'waitqueue' release all locks and call schedule
When someone calls up if it's postive then grab it and go up
```
> [! semaphor utility]
> It's a lock that doens't spin, so you can hold it and sleep/block, It also wastes less CPU time because while your'e spinning. Use them when you need to block. Note if you're using both, then get semaphore first, then grab lock
> More expensive because it needs to call a lock to alter the internal semaphore stuff


## Kernel memory
* Malloc in userspace is all virtual, doesn't corresponid to anything real
* kmalloc responds to actual real physical memory
* Each process has kernel and user address space
* Architecture specific, but generaly the way that the kernel allocates memory is it copies an entry frmo the kernel page table 
* In Arm64 Two registers, user page table base register and kernel page table base register
	* Copy from User in arm64
* The way we do it is we create a virtual address space that corresponds 1 to 1 to real addres space
	* The way it's done is we have a freelist of all free frames of physical memory
* There are macros that convert from pfn to struct page and struct page to pfn because they're well defined
* Two ways to allocate memory: Either allocate pages or allocate non page size units
* **rewatch lecture about how kernel memory allocation works**
* If you want to allocate pages, it'll run alloc_pages -> **struct page**
	* Allocates $2^n$ _contiguous_ pages
	* returns the first frame within a continuous memory
	* Uses powers of two allows page table to preserve really big pages for when someoen needs them (buddy algorithm)
		* Then it merges pairs of free memories when they are next to each other
* Alloc_pages
	* Alloc pages returns physical memory type bestie
* \_\_get_free_pages 
	*  Returns virtual memory space
* **Alloc_Pages_exact (source/mm/page_alloc.c(#L5822))**
	* Returns exact virtual memory space
* kmalloc
	* Returns a "cache" (different from other uses of the word) of pages
	* userspace Malloc asks the kernel for a page and caches that allocation so when the caller asks for more it gives them more of a chunk of the page
	* Kmalloc works the same, gets a cache of pages and gives from them
		* Issue: It may sleep, to wait for smaller granularities of memory be aware and handle
		* Used for **32B->128kB** 
* Use alloc_pages_exact for the shadow page table
* When pages are allocated
	* In the start of a new process
	* On a page fault
		* check do_page_fault arch/arm64/mm/fault.c
			* \_\_do_page_fault
				* handle pte_fault
		* source/mm/memory.c
			* Alloc_page_vma in gfp.h
			* __alloc_pages in page_alloc.c
			* So many functions deep >\_<

### 2 more things
* Huge Pages-> multiple granularity of pages
	* Beyond just 4kb
	* Assume that one page goes down the granuliarity of pfn
	* The way huge pages work is you just go to PMD level, and get a physical frame, which are essentially 512 pages
	* ask prof about the structure of the like page trees, because I thought the pmd level would just be a directory


--- 
# Paging - Nov 9

## Repaso
* Virutal Address -- Pages
* Physical Address - Frames
* Page Table
	* PGD
	* PUD
	  PMD
	* PTE
* Relationship between virtual address and physical address
	* How they're an index of pgd then pud then pmd then pte then offset from the page we reach
> [!more in the pte entries]
> in each PTE there will also be some information about the memory, is it executable, is it writeable, when was the last itme we were written
## Paging the Page Table
### How much space does the page table take up?
* Imagine a 64 bit system (typically implemented as 48 bit address space )
* page size is $2^{12}$
* $2^{36}$ pages
* each entry is roughly 64 bits (8 bytes)
* $2^{39}$ bytes, 550 gigabytes per process, that's a lot 
* Since most address space isn't used, we shouldn't use all the space
	* but since the hardware indexes the address space, it expects a contiguous space of memory
### Solution: Page tables hierarchy
* top level page table **pgd** (page directory) that will point to a page table entry **pte** that will point to a page frame number
* Virtual address space =  |pgd |pte |offset
* This allows the page table to be stored non-contiguously, which means that you only need to allocate the page tables you use
* In practice on most 48 bit address space
	* |pge|pud|pmd|pte|offset
		* The information in each entry is a physical memory address to the next level
	* This makes the TLB the most important, 
	* 9 bytes in a 32 bit address space
* Since page is 4kb (32 bytes), and each entry points to a page, in order to index a page table entry you need 9 bits to index it

## In Linux:
* each task_struct has a memory space `mm struct`
* MM struct stores virtual memory regions as `vm_area_struct`
* Frame number `pfn`
* memory region -> vm_area_struct

## MM struct
* in include/linux/mmtypes.h
* Everything in code is virtual addres sspace
* it's really easy to translate/find phytsical adderss space based on virtual address space
	* macros are typically in archecture specific memory.h, like pHYS_TO_VIRT on line 295 in \_-virt_to_phys
* vm_area_structs are stored in a maple tree mm_mt
	* maple tree is a nice self balancing tree
* do_page_fault in arch/\<archname>/ mm/fault.c
	* the first thing we do is call find_vma, which traverses the mapletree and response if it's valid
	* otherwise it'll do handle_mm_fault in mm/memory.c
* handle_mm_fault
	* \_\_handle_mm_fault line 5000
	* walk thoroiugh the (virtual representation of the ) page table and figure out why it happened
		* is it because we were missing a pte and it needs to be filled? 
		* are we accessing memory that's out of bounds?
		* are we trying to write to read only memory?
		* offset vs alloc
			* line 5017 get pgd
			* alloc will allocate a structure if it doesn't exst, return if it does exist
		* handle_pte_fault
		* in memory.c line l894
		* Pte_offset_map in line 4928
		* get the PTe information
		* Then it'll check if it's anonymous
			* If it's anonymous there's no filename associated iwth it
			* like memory allocated with malloc etc.
			* If you mmap a file into memory
				* I.e. copy them into memory and then write to them and hten write it back into the file
				* that's **file backed memory**
		* Once you get a physical memory address, you need to convert it to virtual to go to the next level
			* This is in offset and malloc
			* When it does bitshifting and anding to select the correct bits
# Paging - Nov 2


- ﻿﻿we know every program has a virtual address space
- ﻿﻿There exists what is called a translation table
	• essentially there is a translation table whose input is a virtual address and the output is a physical address in RAM > like a mapping or array
	- ﻿﻿this is a per process translation table
	- ﻿﻿this mapping is controlled by the Operating System, so it dictates what memory can be accessed by the process
- ﻿﻿If we only map to individual addresses, we'd be very inefficient testing the validity of each adress
- ﻿﻿Instead we have a page table
	- ﻿﻿this is an array of pages, the pages are of fixed size, usually a power of 2
	- ﻿﻿we then deivide our RAM into frames, which are page sized blocks in RAM
	- ﻿﻿our page table converts pages to frames
	- ﻿﻿to convert from a virtual address to physical address, we translate the top order bits of our address which are the page, convert it to the frame number, then glue the frame number to the offset bit in the original address
	- ﻿﻿to simplify, page is a virtual address space, a frame is a physical address space, one maps to the other

The CPU contains a register that has the location of the apge table base register, the operating system will store the physical address of the page table 

* Translation from page table to frame happens within the hardware

 **Kernel  Virtual Memory**
	 Kernel has it's own page table for virtualized memory in the linux kernel
## Translating VM to physical Memory
* The physical memory will correspond to page number and then offset
	* So the process of translating virutal memory is getting the frame number that coresponds to a page number, and then indexing some number of the offfset
* **page size** corresponds to how many bits corresponds to offset, the remaining bits on the left will correspond to page #


## How the Page table gets populated
* First some sysscall will request a certain amoutn of memory, and then the OS will tell you which portion of the virtual address space will be allowed to used by the program
* Os keeps track of starting and ending address (va_start - va_end) of virtual address space, makes sure that it follows along 
*  when the process tries to write memory, it will cause a **page fault** exception (since there is no page that corresponds to the allocated memory)
* the **Page fault Handler** will look at the translation that was tried to be done
	* if the region it's trying to allocated is outside of the predefined range, throw an exception
	* Otherwise, the OS will assign some free frame to the process page table
		* So it allocates space on a page by page basis
	* **principle of locality** it's v likely that if a process accesses one memory location, it''ll probably look in places nearby. Since we 

## Making page table faster - Caching (TLB)
* Introduce a CPU cache of the translations (translation lookasside buffer \[**TLB**])
	* TLB is special purpose on chip fast memory
	* Before going to the trnaslation table, it goes to the CPU cache
# Scheduling Cont. - Oct 31

## Scheduling Classes


### Normal Scheeduler/CFS
* Tries to be fair with finer granuliarity
* assign each task a **virtual time (VT)** 
* $VT_i = \frac{execution\_time}{weight_i}$
* Run lowest virtual time
* Bigger weight jobs increase slower.
### Realtime Scheduler
* has 2 policies (our scheduling class should have two) - sched_RR & sched_FIFO
	* has a bunch of queues from 99 to 0, shcedRR will find the first non empty queue frmo 99 and run that task
		* If it's FIFO, then it will run the task to completion assuming gthere's nothign else in a higher queue torun 
		* if i'ts Round Robin then it'll run the queue like round robin (i.e. hfter timout put it on the end of the queue etc)
		* **note that runscript runs in realtime fifo, expects eveyrthing else to run setschedule on itself to go somewhere else**
> [! Round robin implementation]
> See task_tick_rt in sched rt.c on line 2632 to see how they do timeouts :3 

## Multiple Runqueues
* Solves the bottleneck of multiple CPUs needing to access a single runqueue lock
* But it has some issues
### Load Balancing
* Simple solution: if I'm idle, find a busy CPU and take one of them
	* Issues related to number of tasks, how long each task takes, etc. plus wanting to be fast & efficient
* Main function is scheduling class balance() function
* 
### Priority Scheduling
## Important Functions #linuxcode
> [!CONFIG_SMP]
> anythign with \#ifdef config smp, will only happen if you're on a multiprocessing system, which is likely
> 
* 
* **Check Preempt_cur**
	* Check if this process should be preempted
		* will set tif_need_resched with set_preempt_need_resched()
	* Get's called when it's running, before 
* **task tick**
	* What the scheudler does on a timer interrupt
* **priochange**
	* where you go when shceudling parameters change, for example, when schetscheduler was called
* **switched_to**
	* A task is now being moved into your scheduling class, what do we do
* **update_curr**
	* Opposite, a task has been sent out of your scheduling class
* **Balance**
	* Balance runqueues
* **select_task_rq**
	* What runqueue should I put my task on 

## Scheduling Entity
* Has th estuff that you need to enqueue and dequeue tasks from the runqueue
* For example, if the run queue is a lisut, then the scheduling netity will contain the list head which will be used to put the task on a list

## TIF_NEED_RESCHED
* set when for some reason it needs to call schedule, but it's not on a path where we won't call schedule
* *example* in exit to user mode, before we exit from kernle space to user space, we scheck, and if we need rescheduling, then we'll leave
# Scheduling - Oct 26

* ## What Scheduling Is
	* Queue all tasks
	* pick on eto run
	* Decide for how long
		* (time quantum)
## Basic Scheulders
* FIFO
	* infinite time quantum, keep runnning until ur done
* Priority
	* Every task has some priority assigned to it, the tasks with the highest priority gets picked first
* Round Robin
	* Give time quantum T, and then run everyone for T amount of itme, after put them on the back of the queue
* Shortest Job First actually has the *minimum waiting time* provably
	* Issues: How do you know how long a task will take
	* What if other shorter tasks keep going in front of it
## When do we Schedule?
1. Running task involuntarily gives up CPU
	* #linuxcode : Timer interrupt, preempt (with *TIF_NEED_RESCHED* flag set)
2. Running Task Voluntarily Gives up CPU
	* Blocks or Exits
3. Task becomes Runnable 
	* I.e. A Task was waiting for something and now has woken up
	* #linuxcode : If Preempts enabled, set *TIF_NEED_RESCHED* flag telling it to call schedule
4. Scheduling Parameters change
	* #linuxcode **sched_setscheduler** syscall which will change parameters and then will call **preempt_enable** which will eventually call schedule()

> [! Types of Scheduling]
> *Premptive*: Scheduler interrupts the OS, 1,2,4
> *Non-Preemptive*: Waiting until a task gives up the CPU 2

## Types of Scheduler #linuxcode
* Core DIspatcher 
	* In core.c
* Scheduling Classes
	* Types of schedule functions, like round robin etc.
	* Underlying system asks each class if they have any processes that need shceduling, and selects that

## Schduling Tasks

* Lowest to Highest:
	* Idle Class -> Idle Task
	* Normal (CFS)
	* deadlines
	* realtime scheduler
* When we configure our new shceduling class we should start with it configured below the one
* Processes will get put in the same scheduling class as their parent
## TIF_NEED_RESCHED
* Checked at various points, when a process bumps into that flag it'll call schedule

## Creating Scheduling Task
* Create own Runqueue
	* In struct RQ there is a rq for each scheduling class
* Scheduling Entity
	* Puts tasks in specific Runqueue
* **Task Struct**
	* In Task struct add list head to task struct so they can be put on specific runqueue
* Scheduling Class
	* OOP Bullshit
	* Class with a bunch of methods that are attatched to the class
	* Define function pointers to point to these funcion
	* Defined with **DEFINE_SCHED_CLASS**
		* Example fiels are kernel/sched/\<schedclass>.c
	* Defined in **sched/sched.h**
	* Almost ever class specific functions are going to be done by the underlying dispatcher which takes care of the lcoks, so you don't ahve to worry
		* **1 exception is if you're moving tasks from one CPU to another**, you'll need to get the runqueue locks of the new CPU that you're leaving between
			* To avoid Deadlock: function: **double_lock_balance/double_unlock_balance**
			* Will grab another runqueue lock in the right order
	* Policies
		* Some integer, defined in include/uapi/linux/sched.h L116
			* SCHED_NORMAL : default
			* SCHED_FIFO for fifo scheduling class
			* SCHED_RR for realtime class etc.
		* Make a new sched number Policy
	* NOTE SCHED_INIT THE BUG ON FOR THE DIFFERENT ORDER OF THE CLASSES LINE 9667


## Pick_Next_Task  #linuxcode  
* Most simple case
	* \_\_pick_next_task
		* for each class in the runqueue call the pick next task for each scheudling class and then does pick_next_task of the first one
## Runqueues #linuxcode 
* Most are defined in sche/sched.h
* Each task is only put on one runqueue 

#linuxcode the order of sched class addressees is defined in include/asm-generic/vmlinux/lds.h on line 127


* Use sched_setscheduler to switch a process to a different scheduling class

## Wake_up_new_task #linuxcode 
* First figure out what runqueue to put it on (4729 core.c) select_task_rq
* ## Select_task_rq
	* will then call sched_class specific \select_task_rq tleling it hwere to go
* ## Enqueue_task
	* ### Class specific enqueue_task
		* will eventually put it on the specific enqueue_task


## Important Function fors Task Scheudler
### pick_next_task
### enqueue_task
### dequeue_task

### task_tick
* Occurs whenever you have a timer interrupt
* Look at realtime task tick function in kernel/sched/rt.c l632
	* rt.time_slice is the time quantum
### resched_cur
* Reschedule's the current Task
* DOES SET_TSK_NEED_RESCHED

### set_cpus_allowed
* For now set it to CPU 0
# Midterm Review - Oct 19 
* TODO: LOOK AT HW 1-3 SOLUTIONS
* Practice Exam posted on Ed
* Topics:
	* Waitqueues (focus more on this)
	* RCU locks (R/W)

## Mental Models: #Hardware 
* the Hardware loads instructions at the PC, runs it, and then fails if there's no instruction
* If an instruction is a syscall or causes an exception, switch to kernel mode and switch the PC to the syscall or exception handler and run from there
* If we're in kernel mode and the instruction is an exception return (i.e. the handler is done), then load userpc and switch into user mode
* Every time we do an instruction, check for interrupts
## Mental Models: OS
* Allows multiple programs to run, and maintains control of the hardware
* Abstracts programs with concept of proecess - a thread of execution and virtual address space
### #Processes/Tasks
* Runs user code : User address space
	* User code can only touch user code address space
	* Run user mode unless you get syscall, exception, or interrupt, then call kernel code
* Runs kernel code - Kernel Address space
	* Kernel code touches kernel address space (Kernel address space is shared accross all tasks)
	* If the process is in exits, blocks, and preempts, then it will update the state field to what's relevent then call schedule
	* Then return to user mode 
## OS Bootstrap
1. Start in kernel mode
2. enable virtual adress space
3. config 1st process (ptd 0)
4. enable "kernel mode" (i.e. set up syscall handler so userspace programs can go into user mode)
5. Initialize exception handler
6. Create 2nd process (PID 1)
## #Syscalls
* Functions in the kernel that can be called by user-space
	* Like read, write, fork, waitpid, getpid, signal, sigaction
* Not called directly though, you call the syscall function with a syscall number and maybe some other flags
* the handler will then index into a syscall table to call the specific function 
	* Syscalltable is in a spceial file
	* in ARM it's unistd.h (and update number of total system calls[^1]) and then make the function somewhere else
	* on x86 it's 
	* Function defined in SYSCALL_DEFINE(){}
* In Syscalls we don't trust the user or, so we use copy_to_user and copy_from_user

## #Interrupts
* Asynchronous events
* External hardware event begins it
* interrupt handler does not call schedule
* however it can wake up a process

TODO:
## CONCURRENCY/ SYNCHRONYZATION, 
* Atomic Types
* Spin Locks
* Spin Locks + Wait queues to allow something to lock
	* Let go of lock while on wait queue
* Reader/Writer Locks
* RCU

ACCESS CODE: 4729


[^1]: __NR_syscalls




