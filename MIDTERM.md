

# Specific Code Things
## Locks & Interrupts

### RCU Lock
rcu_read_lock()
rcu_read_unlock()

### Runqueue Lock
rq = cpu_rq(cpu);
struct rq_flags(&rf);
rq_lock(rq,&rf)
context_switch(prev,next)  **also disables runqueue lock**
### IRQ Lock
local_irq_enable()
local_irq_disable()

### Raw_Spin_Lock
raw_spinlock_t pi_lock (in task_struct);
raw_spin_lock_irq(&p->pi_lock);
raw_spin_unlock_irq(%p->pi_lock)







## Syscalls
ARM
* in linux/kernel/include/uapi/unistd.h
* \#define \_\_\nr_\<NAME>, sys_FUNCTION_NAME
* \_\_SYSCALL_NR_\<NUMBER>(\<FUNCTIONNAME>)
- SYSCALL_DEFINE(number,name,kernel_name)
- update \_\_NR_SYSCALLS
X86 
* in linux/kernel/arch/x86/syscalls_64.tbl
* add newline as follows
	* NUMBER COMMON NAME SYS_NAME


then in another function
SYSCALL_DEFINE\<N>(\<NAME_SYSCALL>,\<ARGS>){}
# IMPORTANT FUNCTIONS
## Syscalls
* DEFINING SYSCALLS
	* **X86**: arch/x86/entry/syscalls/syscall_64.tbl
		* NUMBER | COMMON | NAME | SYS_NAME
	* **ARM**: 
		* in UNISTD.H
			* \#DEFINE \_\_NR_\<NAME> NUMBER
			* \_\_SYSCALL(\_\_NR_\<NAME>, SYS_\<FUNCTION_NAME>)
			* INCREMENT NR_SYSCALLS
	* access_ok(Pointer, size)
		* Make sry functions r good
	* copy_to_user(destsource,dest,sizeof)
* CALLING SYSCALL
	* INCLUDE UNISTD.H
	* ROOT/INCLUDE/LINUX/\<FILENAME>.H
	* SYSCALL_DEF\<ARGC>(<FUNCTION,NAME>,\<ARGS>)
## TASK STRUCT
* real_parent
* parent
* pid
* tgid
* state
* exit_state
* list_head children
* list_head sibling
* getTID(return PID)
* getPID(returns TGID)

## WAITQUEUES
# Questions


Questions are like more about specific functions and code rather than concepts
- try_to_wake_up()
- wake_up_new_task()
- \_\_schedule()
- wake_up_all()
	- What does it mean to wake up a process on a waitqueue

## Wait_event_interruptible
* Takes two variables, the head and condition
* which will then call \_\_wait_event to be interruptible and call schedule
* in prepare_to_wait_event check if you've received any signals
* If it's interruptible check there are any signals, and if so then get interrupted (go to  whatever the interrupted thing is), otherwise call schedule()

Turn off IRQs (so nothing can interrupt you while you're holding a lock)
# Overview of Hardware & OS 
* the Hardware loads instructions at the PC, runs it, and then fails if there's no instruction
* If an instruction is a syscall or causes an exception, switch to kernel mode and switch the PC to the syscall or exception handler and run from there
* If we're in kernel mode and the instruction is an exception return (i.e. the handler is done), then load userpc and switch into user mode
* Every time we do an instruction, check for interrupts
* Allows multiple programs to run, and maintains control of the hardware
* Abstracts programs with concept of proecess - a thread of execution and virtual address space
# Processes/tasks
#processes
> Lecturees 9/15- 10/10 textbook ch 26-
* Runs user code : User address space
	* User code can only touch user code address space
	* Run user mode unless you get syscall, exception, or interrupt, then call kernel code
* Runs kernel code - Kernel Address space
	* Kernel code touches kernel address space (Kernel address space is shared accross all tasks)
	* If the process is in exits, blocks, and preempts, then it will update the state field to what's relevent then call schedule
	* Then return to user mode 
* Goes back to kernel for syscalls or from #interrupts (hardware signal that says go back to kernel space and execute some other code)
* Examples of Hardware interrupts
	* Timer interrupt, after a certain amount of time go do something else
	* I/O interrupt, stop doing anything else and process this keystroke/mouse movement
* Interrupts can be disabled in Kernel Mode, (privileged instruction, only available in CPU mode)
* **Tasks finish in exit.c in exit()**
	* Clears all the fields in task_struct, then calls do_task_dead which calls schedule() 
## Process  Lifecycle
> Lecture 9/26/23
1. Created with Clone
2. Put on runqueue, set as TASK_RUNNING
3. Starts running by other task calling schedule()
4. At some point will then call schedule() and something else will run
	1. Like if it's done
	2. or timeout interrupt 
	3. or if it's blocked
	4. Or it receives a *Signal* (signal could make it block or stop)
		1. It sees it receives a signal when it's in the kernel, i.e. at the beginning or end of it's life, or after an interrupt 
5. When it's done completely it goes into zombie mode, it won't run any more code, but it hangs around in case someone is waiting for it
	1. When it's a zombie it's address space will be used for other processes, 
6. then the task becomes dead once someone calls wait on it
## Task_Struct
> Lecture 9/21/23
* Data structure for Processes in linux
* Has list of Chidlren, address space, etc.
* thread field wil be architecture specific
* has cpu_context, which keeps track of the PC 
	* Note that this isn't actually the definition of the PC, just a variable, which then other arch/specific stuff will implement
	* in ARM you'd have to set the linkr egister to the pc then call return to go to the pc
* Real_parent: Whoever created this task
* Lists
	* List of every Task
	* List of every child
	* List of every sibling 
* Task Structs are allocated on heap in the kernel
	* It doesn't live within any individual process
* Exists as part of Process Tree


> [!Hardware Protection Mechanisms]
> So that processes can't exist out side of their context, we have the following hardware protection mechanisms
> Kernel Mode/UserMode ->
> Hardware Interrupts/Timer Interrupt ->
> Address Mapping

## Threads
>[!Definition]
>Thread of execution within a process. Threads amount to CPU States, and they share Address Space
#threads 
### Functionality of Threads
* Faster than new processes because they don't need a new address space
* Since modern CPUs have multiple cores, u can have multiple parts of a process run at the same time on difference CPUS (as different threads)
> [!Caveat] 
> There are system calls that allow you to make the CPU make two processes share an address space, but that's not default

* in linux You can make a new thread by doing clone with the CLONE_THREAD flag
* in Linux tasks relate to both threads and processes 
* For threads, task_struct's ->mm_struct[^2] will point to another task struct's mm_struct
### backwards-compatibility edgecases
* fork() will only create new thread
* getpid will get the thread_group_leader's PID
* parent of any thread will point to the process that created the thread group leader

### Features of threads
* getPID() will return the TGID
* getTID() will return the PID i.e. the unique ID of the thread
* Access things through macross
* **Thread_group** 
	* Every task with the same address space is a Thread_group
* **Thread_group_leader**
	* whatever task created the address space is the thread_group_leader
	* Tleader has a list of the rest of a group
* Every Thread has the same parent (parent of the process, not TGLeader)
* **non_thread_group_leaders will not be in the list of tasks or in the sibling or child lists**
* **Threads Autoreap** -> They go straight from task_running to exit_dead
* If a thread calls fork, pretend the Thread_group_leader called fork

### Sharing Address Spaces
* Example: If there are two threads that both at roughly the same time increment and decrement the number, depending on the assumbly code (when they loaded the variables) it may end up differently 
* To fix this: we do Locks!




[^2]: address space
# OS Bootstrap
1. Start in kernel mode
	1. Boot up in ROM
	2. set PC to UEFI
2. UEFI: Configure Hardware/enable virtual adress space
	1. Configures frame Buffer
3. config 1st process (ptd 0)
	1. Defined in init_task.c
4. Initialize Scheduler
	1. Initialize exception handler
	2. Initialize Scheduler
5. Forks 1st process (PID 1) \[System_MD\], creates a runqueue
6. Becomes idle, calls schedule, switching to process 1
7. 1st process does File init (like mounting the file system )
	1. "Exec the init program" (outside of kernel space)
8. Init starts other processes
9. return to Kernel for syscalls
# #Syscalls

* Functions in the kernel that can be called by user-space
	* Like read, write, fork, waitpid, getpid, signal, sigaction, exec
* Not called directly though, you call the syscall function with a syscall number and maybe some other flags
* the handler will then index into a syscall table to call the specific function 
	* Syscalltable is in a spceial file
	* in ARM it's unistd.h (and update number of total system calls[^1]) and then make the function somewhere else
	* on x86 it's syscall.tbl
	* Function defined in SYSCALL_DEFINE(){}
* In Syscalls we don't trust the user or, so we use copy_to_user and copy_from_user

## How to write a Syscall
* Edit syscall table: unistd.h in ARM and in x86 it's sycall.tbl
* then the Kernel, run SYSCALL_DEFINE_N( <\Number>)
	* where N is the number of arguments the syscall can have
* Single Variables/primitives
	* Put User
	* Get User
* Block of Data
	* Copy_to_user
	* copy_from_user
* access_okay() macro that checks that pointers point to accessible memory space
* Return:
	* Always an integer
	* if there's an error then return -1 and errno is set
* Errors
	* EFAULT
	* ENOMEM
	* EINTR
	* ENOSEARCH



# Scheduler 
> 10/10 Lecture, Os Ch 9 and 10(?)

## #Runqueue Data Structure
* 1 Per CPU, global variables has list of processes that the Scheduler than picks to run on the CPU
* Has it's own Special lock
## Scheduler Initialization Process
* Initialized by Process 0 with sched_init()
* Initializes runques and sets all their states to unlocked
* Sets PID 0 task to Idle
## Fork
* When PID 1 is made by fork, it runs copy_process, which will call 
* wake_up_new_task
	* Put task on runqueue by grabbing the runqueue lock, enqueueing it with *activate_task*, and then release lock
## Blocking Tasks
* **High Level**: Get's removed from runqueue and put on a waitqueue, then it's state is interrupted, then it called schedule()
* First we create waitqueue
* calls do_wait, which adds ourselves to the waitqueue and changes our state, then calls schedule
### #waitqueue s  
> lecture 10/12/23
* If, for example a thread blocks, then you want to put it on a waitqueue! (waitqueue_entry)
* First set up initialize waitqueue head init_waitqueue_head() (from wait.h)
* wait_event_interruptible(wq_head,condition)
	* Sleep until a condition gets true, on waitqueue wq_head
* Waitqueues will sleep until someone else wakes them up 
	* You can't wake up specific tasks from a waitqueue, when you call wakeup on a given waitqueue it will try to wake up each element in the waitqueue until one wakes up
* Wake up Options
	* Wake_up_all
		* Wake up everyone on the waitqueue, if they're all waiting on the same condition
	* Wake_up
		* Wake's up one task
	* Wake_up_interruptible
* Waitqueues have a field called field_func_entry, the function to run when the event occurs
* **NOTE**: you can't call wait_event_interuptible with a lock, since it'll call schedule, so you should unlock, call wait_event_interruptible, and then grab the lock again
* hold on: When you make your own waitqueue, it'll eventually in wait_event get put on a different waitqueue made for that specific conditional, so like everything waiting for that specific conditional wiill get woken up when you get woken up :3
#### Sync w/o race conditions
* 2 Condition Variables- _cv.wait_, and _cv.signal_
	* cvwait(x):
		* Add to smth to wait queue, unlock, call schedule, and then lock
			* This does everything we can with the lock before callign the schedule
	* cv.signal(p)
		* if the wait_q isn't empty
		* then remove p from teh waitqueue
		* then wakeup(p)
```
Program 1                     Program 2
Lock(x)                       lock()
do work                       do work
cv.wait(x)                    cv.signal
work                          unlock
unlock
```
* In this version, the waitqueues are only used for waiting, the actual checking of conditions is done by cv.signal within the lock, so that 

* E.X. waitpid
	* when child dies, it does do_notify_parent whcih does \_wake_up_parent which will call try_to_wake_up, which will wake up an old task and put it back on the runqueue
* 
## \_\_Schedule() Function
* First disable Interrupts
* Grab Runqueue Lock 
*  Run Pick_next_task to pick the next task
* If the task's state is not running (I.e. It's blocking)
	* Remove it frmo the runqueue, it's no longer on the runqueue, just on a waitqueue
* Then run the context switch to return from fork (architexture specific)
	* Context_switch calls
		* Prepare task-switch
			* prepare task switch and runqueue things
		* Switch_task
			* Swap CPU context and save state of CPU into task-struct
	* Return from Fork (if new process)
* Finish_task_switch
	* lets go of runqueue lock and enables interrupts 
* Preempt count: incremented by us holding a lock
	* we can't schedule anything until preempt counter is 0
## #Interrupts
* Asynchronous events
* External hardware event starts it
* however it can wake up a process
* Interrupt handler has a lot of priveleged access, if a process is interrupted **the process is not running the interrupt code**
* Therefore, when you are running interrupt code you: 
	* Cannot block
	* cannot calls chedule
* When ur done you call some kernel code which brings u back to usermode
	* Set a flag on a task that says that it should be scheduled, and then when the interrupt is done then the reschedule will be called
> [! Interrupts vs Preemptions]
> An Interrupt is the hardware telling the CPU to go somewhere else, Preemption is just when the kernel swithces to another program. Disableing preemption means that if schedule() is called, then the kernel will not actually context switch into another process


## Context_Switch
* Change registers
* save state of existing context 
* swap the task (finish_task_switch)
* Lock is obtained by the process about to switch, then released by the next one

# CONCURRENCY/ SYNCHRONYZATION, 
## Locks - 
>[!Definition] 
>If you need to edit a critical section of code, you have to grab a lock first,. If someone else has that lock, you're going to wait until the lock is free

### Features of locks

1. Mutual exclusion one thing touching critical section at a time)
2. Progress: If nobody has the lock you get the lock
3. Bounded Waiting- Guaranteed that at some point you will run
### Spin Lock
* CPU "spins" (keeps checking if it can get the lock)

### test-and-set
* Change the value of the lock, and also returns the old value
* Operation is performed atomically, it does both at the same time
* that way, you only know the lock is free the exact moment you set it to a new value, allowing you to avoid issues

### Ticket Locks
* Increment and return old value to find out what your turn is, when the counter hits your number you can get the lock, otherwise spin
* this allows for Bounded Waiting

### Reader/Writer Lock

different locks for reading and writing. Readlock allows multiple people to share the readlock (not mutually exclusive with other readers), however is mutually exclusive with wrier, if someone has the write lock, you can’t get the read lock. 
#### RCU lock synchronize Implementation:

- write lock and unlock are the same, just use spinlock
- for read_lock
	* mutually exclusively (using another spinlock) increment a counter and grab the lock if there are no other people reading it for the unlock decrement the count and if there is none left then let go of the spinlock

Read-Write lock is good except in the case of like large machiens with a lot of processes, since everyone still needs to grab the lock (biased towards reader)
#### RCU synchronize
- Called after any writing is done
- Makes sure that every single CPU/process has called schedule, which means that nobody is looking at the old version, since you’ve already written


> [!Avoid deadlock!!:]
>  if you have mutual exclusion, hold and wait, no preemption, and circular wait (circular wait meaning a program is waiting for itself, like trying to grab a lock that you already have, etc.)

*Deadlock avoidance*
1. Don’t activate multiple locks
2. If you need multiple then do it in some order, so like if you need lock 1 and 2, let go of lock 1 do stuff with lock 2, then grab lock 1 again

## Atomic Types
- Allows you to do CPU arithmatic, and bitwise operations atomically (atomic_add, atomic_read, etc.)
- This means that they need to be e architecture-specific
- Key for Test_and_Set
- [Types](https://elixir.bootlin.com/linux/v6.1.11/source/include/linux/atomic/atomic-instrumented.h#L39 )


  


[^1]: __NR_syscalls


# Homework 1
## Differences between shell w syscalls
* Main
	* MunMap instead of free
	* mmap instead of malloc
	* Input() instead of getLine()
* print_prompt
	* `output(fd,<printf>,<formatting>)`Instead of `fprintf()`,
# Homework 2
>[! #Question]
>What's the difference between copy_to_user and put_user
* Do_ptree
	* create Kfifo with kfifo_alloc
	* get root task_struct with get_root()
	* enqueue root process with kfifo_in
		* Update count to include threads and total tasks include
	* BFS loop
		* pop top from KFIFO
			* Keep track oflevel with counter and set new_level_flag to 1
			* increment level counter when we're at the level_root_proc i.e. the root of a new level since we just popped the root node of whateve rleeve we're at
			* if level_root_flag is one then whatever the first child we see is becomes the new level_root (i.e. it's children define the new level)
	* Loop through each thread of the threadgroup
		* Loop through every sibling in the the thread
			* According to notes this is "necessary to catch children created by fork from within a pthread"
				* **Ask what this means**
			* 
* get_root()

# Homework 3
