Design Document for Project 2: User Programs
============================================

## Group Members

* Hadrien Renold
* Mark Sun
* Keshav Potluri
* Yichen Ruan 

# Task 1: Argument Passing
## Data structures and functions
In file [/pintos/src/userprog/process.c](/pintos/src/userprog/process.c):
```c
/* Call the function to tokenize file_name */
bool load (const char *file_name, void (**eip) (void), void **esp) {}

/* Extract the argc and argv from file_name */
static void tokenize (const char *file_name, char *argv[], int *argc) {}

/* Push the argv and argc to stack. */
static bool setup_stack (void **esp, char *argv[], int *argc) {}
```
## Algorithms
We will first pass the file_name to the `tokenize` function that we created which will split the string on spaces and create a list of arguments. This list of arguments will make up our `argv` and their count would indicate `argc`. We will do this operation within the `load` function which is called within `process_execute`. The next step is to pass on these arguments to the process. This can be done by pushing the values onto the stack while it is being created for the process. This happens within `setup_stack`. We will push the values to the stack in the following order:
 1. Push the values of elements of argv
 2. Push the addresses of these elements from right to left and a sentinel null pointer.
 3. Push argv (address of argv[0])
 4. Push argc
 5. Push a fake return address
Once the values are pushed to the stack, the arguments are read by the process in its execution.

## Synchronization
There are no significant race conditions observed in this part. Although a race condition exists for the `file_name`, it has been already handled in `process_execute` by creating a copy of the `file_name`.

## Rationale
The way we could pass arguments to a process is by simply pushing the arguments onto its stack. Therefore, we decided to modify `setup_stack` for passing the arguments to the process. We also decided to split `file_name` into `argc` and `argv` within load to minimize the changes in existing function signatures.


# Task 2: Process Control Syscalls
## Data structures and functions
In file [/pintos/src/threads/thread.h](/pintos/src/threads/thread.h):
```c
/*** ADDITIONS ***/
struct child
  {
    struct thread* t;
    int status;
    struct semaphore sema;
    struct list_elem elem;
  }

/*** MODIFICATIONS ***/
struct thread
  {
     ...
     struct lock lock_the_children;
     struct list children;
     struct status *self;
     struct lock self_lock;
  }
```
In file [/pintos/src/threads/thread.h](/pintos/src/threads/thread.c):
```c
/*** MODIFICATIONS ***/

/* Modified to accept a status. All calls to thread_exit will be updated with appropriate exit statuses either EXIT_FAILURE or EXIT_SUCCESS.  */
void thread_exit(int status);

```

In file [/pintos/src/threads/vaddr.h](/pintos/src/threads/vaddr.h):
```c
/*** ADDITIONS ***/

/* Read a variable of size SIZE pointed to by the virtual address UPTR. Handles reading over page boundary.
Returns a physical address of a buffer containing those bytes, or NULL if the UPTR is invalid. */
void* ustack_read(void *uptr, size_t size);

/* Read a string at the virtual address UPTR. Handles reading over page boundary. Reads up to maximum one page.
Returns a physical address of a buffer containing the string, or NULL if the UPTR is invalid. */
void* ustack_read_str(char *uptr);

/* Validates a virtual address, checking that it is not NULL, that the address is in user memory and that it is valid. Called by ustack_read and ustack_read_str. */
bool validate(void *va);
```

In file [/pintos/src/userprog/syscall(.c/.h)](/pintos/src/userprog/syscall(.c/.h)):
```c
/*** MODIFICATIONS ***/

/* Will modify to call the syscall_functions below. */
static void syscall_handler (struct intr_frame *f);

/*** ADDITIONS ***/

/* Increments i and returns it to the user. */
int syscall_practice(int i);

/* Terminates Pintos. */
void syscall_halt(void);

/* Terminates the current user program. Returns status to kernel. */
void syscall_exit(int status);

/* Runs the executable given in cmd_line. */
pid_t syscall_exec(const char *cmd_line);

/* Waits for a child process pid and retrieves the child's exit status. */
int syscall_wait(pid_t pid);


```
In file [/pintos/src/userprog/process(.c/.h)](/pintos/src/userprog/process(.c/.h)):
```c
/*** MODIFICATIONS ***/

/* Explained below. */
int process_wait (tid_t);
tid_t process_execute (const char *file_name);
```


## Algorithms
All of the syscalls end up with a call to the `syscall_handler` which does the following:
- read the syscall number from the user stack using `ustack_read`
- read the number of arguments needed for the given syscall using `ustack_read` or `ustack_read_str`
- call the appropriate function is the stack reads succeed or else kill the user process by calling `thread_exit(EXIT_FAILURE)`

The following describe the behavior of the functions called by the syscall handler:
### int syscall_practice(int i);
- set the interupt frame eax to i + 1

### void syscall_halt(void);
- calls `shutdown_power_off()`

### void syscall_exit(int status);
- reads the exit status from the user stack and calls `thread_exit` with that value
- the rest is already implemented

### pid_t syscall_exec(const char *cmd_line);

In `process_execute`:
1. We dynamically create a `struct child` and add it to the list of children in the parent process.
2. When creating the thread, we pass in two arguments. The first is a pointer to the child's index in the parent's list, and the child thread will set `self` to this pointer. We also pass in the address of the `struct child` we dynamically allocated as another argument.
3. Initialize the semaphore to 0 and set status to 0 in the `struct child`
4. Call `sema_down` on the corresponding semaphore

In `thread create`:
1. We will modify `thread_create` such that the child thread will set the `struct thread *t` in the `struct child` that was passed in to be a pointer to itself.

In `load`:
1. We make a slight modification to call `sema_up` on `self`'s semaphore after the line `done:`


By calling `sema_down` before we return, we ensure that the parent process cannot return from the exec call until it knows that the child process has loaded or not. This is because the child calls `sema_up` after the `done:` in `load()`. By this point the child process will have loaded successfully or failed. Only then can the parent process proceed past the `sema_down` line and return.

### int syscall_wait(pid_t pid);
The `syscall_wait` calls `process_wait` with the prior validated arguments. `process_wait` relies on the data structures described above. Each thread has a list of `children` which stores for each child, a pointer to the child's `struct thread`, its `status`, and a semaphore. Storing on the parent the exit_status allows to persist the exit status even once the child has terminated (and hence freed its memory).

When the parent calls `wait`, it acquires `lock_the_children`, attempts to verify that the `pid` is part of its children, and releases the lock after iterating through all the children. If it finds a corresponding child, it calls `sema_down` on the corresponding semaphore. Once it is woken up that means the child has terminated and filled in its `status` in the `child` element, thus the parent can read this status and return to the calling process. It can then acquire `lock_the_children`, remove the `child` element from the `children`, free the memory, and then release `lock_the_children`. This way on if the parent calls wait again on the same child, the child will not be present anymore in the list of children and we can return -1.

When the child terminates it sets it calls `thread_exit` with the appropriate status. This happens at three places, either in the function `exit` for normal exit whence the status is set to the user defined status, or in function `kill` of 'exception.c' if the process is terminated by the kernel, whence the status is set to the CPU error status, or in the syscall handler if there is an error.
In function `thread_exit` we follow the `self` pointer to update the `status`, acquiring the lock to do so.
The child then calls sema_up to wake up the parent if it is waiting or to set the semaphore to 1.

If the parent calls wait before the child has finished executing then then parent will be put on the semaphore's waiting list and put to sleep. If the child has already returned, the semaphore value will be 1 and the parent will return immediately.  

If the parent terminates before its children we first acquire `lock_the_children`. We then walk through the `children` and for each child we acquire the `self_lock` and set the `self` to NULL, then release the `self_lock`. We then free all elements of the `children` list, and then release `lock_the_children`.


## Synchronization
The first shared variable is the `status` field of the `child` element, which is being accessed from the parent for reading and the child for writing. However, in both the `exec` and `wait` cases the parent is waiting until this value has been updated, and hence there is no risk of concurrency with read and write.

The second shared variable is the pointer `self` which the child wants to follow when updating it's status and checking wether it is an orphan, and the parent wants to update when it is terminating so as to remove all pointers to itself. The lock enables to have one of those operations at the same time. Moreover because all the operations are simple ones (with no wait) there is no risk of deadlock.

`children` list: The third shared variable is the `children` list, which is shared between the parent process and any children it has. The children have a pointer to their corresponding entry in the list. The only times when this list is edited is when the parent process exits, if the parent process returns from `process_wait`, or if the parent calls `process_execute`. To make sure only one thread is writing at a time, we try to acquire the `lock_the_children` in each of these cases. Furthermore, we also iterate through the list when we wait to see if the tid is a direct child, so we acquire the lock before iterating so that the list is not being written to or deleted when we are doing so.

## Rationale
We considered having a global list or hashtable to store a mapping of parent to children but that would have required a lot of inefficient bookkeeping and traversing of the data structures. We have some difficulty finding relevant names for our variables which are not as explicit as we would like (e.g `self`). A advantage of our design is that we can reuse the same data structure for the exec and wait call to check the status of the child. Because we have a separate struct for the child element we can extend it easily to add other fields which would be required for other features.



# Task 3: File Operation Syscalls
## Data structures and functions
In file [/pintos/src/userprog/syscall(.c/.h)](/pintos/src/userprog/syscall(.c/.h)):
```c
/* Mutex lock for the entire file system */
struct lock filesys_mutex;

/* Data structure for each open file */
struct open_file
{
  /* File descriptor */
  int fd;
  /* file struct in file.c */
  struct file *_file;
  /* list_elem to be added to a thread's open_file_list */
  struct list_elem ofelem;
}

/* Dispatch syscall by switch statement */
static void
syscall_handler (struct intr_frame *f UNUSED)
{
  switch (argv[0]){
    case: SYS_CREATE
      ...
  }
}

/* Validate the entire buffer space */
bool validate_buffer (void *buffer, unsigned size);

/* Close all files in open_file_list before actually exiting */
void exit (int status);

/* Initialize a new open_file_list for child process */
pid_t exec (const char *cmd_line);

/* Implement the file syscall functions */
bool syscall_create (const char *file, unsigned initial_size);
bool syscall_remove (const char *file);
int syscall_open (const char *file);
int syscall_filesize (int fd);
int syscall_read (int fd, void *buffer, unsigned size);
int syscall_write (int fd, void *buffer, unsigned size);
void syscall_seek (int fd, unsigned position);
unsigned syscall_tell (int fd);
void syscall_close (int fd);
```

In file [/pintos/src/threads/thread(.c/.h)](/pintos/src/threads/thread(.c/.h)):
```c
/* Add members to thread struct */
struct thread
{
  ...
  /* Each thread keeps a list of the open files it owns */
  struct list open_file_list;
  /* The maximum fd a thread has */
  int max_fd;
}
```

## Algorithms
a) Before actually calling any file system functions, we will validate the arguments using methods of Task 2. Especially, we will validate the buffer before calling `read()` and `write()`. We also make sure the file name is no greater than 14 characters before calling `create()`, `remove()` and `open()`.

b) A global lock `filesys_mutex` is used to synchronize all file system operations. Thread may be blocked when it tries to access the file system. The lock will be released as soon as an error is encountered.

c) When `open()` is called, an `open_file` struct is allocated to record the `fd` and the `*file` pointer. This struct is inserted into the `open_file_list` of the thread. We also keep a variable `max_fd` for each thread to keep trace of the maximum `fd` number owned by the thread. So the new `fd` is simply `++max_fd`.

d) When `filesize()`, `read()`, `write()`, `seek()` and `tell()` are called. We go through the `open_file_list` to find the `open_file` with target `fd`. Then we can use the `*file` pointer to conduct the corresponding file system level syscall. If the `fd` is not existing in the list, we terminate the process (after closing all open files).

e) When `close()` is called. We also traverse the `open_file_list` for the desired `fd`. If it is found, we remove it from the list, call the file system level `close()`, then free the `open_file` struct.

## Synchronization
The only shared resource in this part is the file system.

As is suggested in the specification, we use a global lock to synchronize all file system operations. A lock is acquired before any file system level syscall, and is released before the function returns, or any error happens. Since no two threads can operate on the file system concurrently, we make sure that the file syscalls are thread-safe.

## Rationale
Instead of keeping the fd-list in the thread level, we can also implement it in the kernel level. However, this will make the data structure more complicated. We would have to keep all the `fd`, `pid` and `*file` pointer, which will make the list operations less efficient. Worse still, the list itself could become a shared resource, implying more overhead work for synchronization. By keeping the list in the thread, the length of the list is shorter for every thread. Thus we have higher efficiency for searching, inserting and removing.

If we want to implement more sophisticated synchronization, however, we may still need some global bookkeepers. But this design is sufficient for current requirements.

# Design Document Additional Questions

## Question 1
The test `sc-bad-sp.c` uses an invalid stack pointer while making a sys call. In the line 18, the code segment `$.-(64*1024*1024)` refers to an address that is 64MB below the current assembling address (referred to by '.') and the `movl` moves the address to the stack pointer, making it an invalid stack pointer. This is followed by the invoking `int $0x30` for a syscall. Since the call expects a 32 bit word at the stack pointer to identify the system call number, it tries to access the memory location. Since the stack pointer this memory location is outside the bounds of the user program virtual memory, it should not access this memory, and the syscall should exit with (-1) code.

## Question 2
The test `sc-bad-arg.c` uses a valid stack pointer while making a sys call but the stack pointer is too close to the page boundary and the syscall arguments are located in an invalid memory. In the test, at line 14/15, first the address `0xbffffffc` is moved to the stack pointer. This memory location is very close to the `PHYS_BASE`, located at `0xc0000000`. Next the syscall number (`SYS_EXIT`) which is a 32 bit word is pushed to the stack, making the syscall number spread into the invalid memory (till `0xc000001c`, going above `PHYS_BASE`). This is followed by the invoking `int $0x30` for a syscall. Since the argument for the syscall, which is the syscall number, extends into the address space for the kernel and therefore the invalid memory, the sys call should not execute.

## Question 3
The current test suite has tests specified for the filesystem calls `read`, `write`, `open`, `create` and `close`. However, the suite does not have any tests defined for `remove`, `seek`, `tell` and `filesize`. For remove, just various tests can be created like : remove should not execute for bad pointers, remove should be able to execute for files whose name spans through multiple pages, remove should execute for empty file names, remove should not execute for an already removed file etc. Similarly, different scenarios can be analyzed for `seek`, `tell` and `filesize`, where we test for incorrect fd, tests with STD_IN and STD_OUT, tests on files spanning across pages, etc. Apart from these, there are plenty of other tests that can be added for the file system functions as a whole to make our test suite more robust (For e.g. test for closing of all open files when a process exits).


## Question 4 : GDB Questions
### 1)
name: main

address: 0xc000e000

other threads:
#0: 0xc000e000 {tid = 1, status = THREAD_RUNNING, name = "main", '\000' <repeats 11 times>, stack
= 0xc000ee0c "\210", <incomplete sequence \357>, priority = 31, allelem = {prev = 0xc0034b50 <all_list>, next = 0xc010402
0}, elem = {prev = 0xc0034b60 <ready_list>, next = 0xc0034b68 <ready_list+8>}, pagedir = 0x0, magic = 3446325067}

#1: 0xc0104000 {tid = 2, status = THREAD_BLOCKED, name = "idle", '\000' <repeats 11 times>, stack
= 0xc0104f34 "", priority = 0, allelem = {prev = 0xc000e020, next = 0xc0034b58 <all_list+8>}, elem = {prev = 0xc0034b60 <
ready_list>, next = 0xc0034b68 <ready_list+8>}, pagedir = 0x0, magic = 3446325067}

### 2)
#0  process_execute (file_name=file_name@entry=0xc0007d50 "args-none") at ../../userprog/process.c:36
```c
sema_init (&temporary, 0);
```
#1  0xc002025e in run_task (argv=0xc0034a0c <argv+12>) at ../../threads/init.c:288
```c
process_wait (process_execute (task));
```
#2  0xc00208e4 in run_actions (argv=0xc0034a0c <argv+12>) at ../../threads/init.c:340
```c
a->function (argv);
```
#3  main () at ../../threads/init.c:133
```c
run_actions (argv);
```

### 3)
name: args-none

address: 0xc010a000

other threads:
#0: 0xc000e000 {tid = 1, status = THREAD_BLOCKED, name = "main", '\000' <repeats 11 times>, stack
= 0xc000eebc "\001", priority = 31, allelem = {prev = 0xc0034b50 <all_list>, next = 0xc0104020}, elem = {prev = 0xc003655
4 <temporary+4>, next = 0xc003655c <temporary+12>}, pagedir = 0x0, magic = 3446325067}

#1: 0xc0104000 {tid = 2, status = THREAD_BLOCKED, name = "idle", '\000' <repeats 11 times>, stack
= 0xc0104f34 "", priority = 0, allelem = {prev = 0xc000e020, next = 0xc010a020}, elem = {prev = 0xc0034b60 <ready_list>,
next = 0xc0034b68 <ready_list+8>}, pagedir = 0x0, magic = 3446325067}

#2: 0xc010a000 {tid = 3, status = THREAD_RUNNING, name = "args-none\000\000\000\000\000\000", stac
k = 0xc010afd4 "", priority = 31, allelem = {prev = 0xc0104020, next = 0xc0034b58 <all_list+8>}, elem = {prev = 0xc0034b6
0 <ready_list>, next = 0xc0034b68 <ready_list+8>}, pagedir = 0x0, magic = 3446325067}

### 4)
```c
tid = thread_create (file_name, PRI_DEFAULT, start_process, fn_copy);
```
### 5)
The line is: 0x0804870c

### 6)
This time btpagefault gives:

#0  _start (argc=<error reading variable: can't compute CFA for this frame>, argv=<error reading variable: can't compute
CFA for this frame>) at ../../lib/user/entry.c:9

### 7)
Because the argument passing has not yet been implemented. The function `_start(int argc, char *argv[])` can't correctly read arguments `argc` and `argv` from the stack.
