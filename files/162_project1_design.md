Design Document for Project 1: Threads
======================================

## Group Members

* Hadrien Renold
* Mark Sun
* Keshav Potluri
* Yichen Ruan 

# Task 1: Efficient alarm clock
## Data structures and functions  
In file [/pintos/src/devices/timer.c](/pintos/src/devices/timer.c):  
```c
struct list timers;  
struct timer_elem  
{  
	struct semaphore timer_sem;  
	int timer_end;  
	struct list_elem elem;  
};
```
`timers`is a global variable list of `timer_elem` which keep track of the different timers set. This list is ordered in ascending order based on `timer_end`.

## Algorithms
We will modify `timer_sleep` such that when called in a thread it will:  
 1. creates a new semaphore  
 2. creates a new `timer_elem` with `timer_sem` initialised with this semaphore and `timer_end` the absolute number of ticks at which the timer is done, ie the tick count for now + the number of ticks to wait  
 3. This `timer_elem` is then inserted in order in the list `timers`  
 4. The semaphore state is set to "Down"
 5. Once the semaphore returns, the created `timer_elem` is removed from the list.  
Note: any operations on the global list timers will have interupts disabled for the duration of that operation to avoid race conditions while editing the list.

We will modify `timer_interrupt` to step through the list `timers`. For all elements `timer_end` < current_ticks.it will "Up" the semaphore, thus waking up the corresponding thread which will be ready to execute again.

## Synchronization
The shared resources in this problem are the elements of the list `timers`. By disabling interupts when modifying this list we make sure that changes made to it are atomic.  There is no dynamic memory allocation since we are using the implementation of doubly linked lists so there's no need to free memory.

## Rationale
This design guarantees that there is no busy waiting by using semaphores to wake up the thread. There might be a more efficient way that uses only one semaphore but we were unable to come up with such a solution. This design could be a litle wasteful as there is a one to one mapping from threads to semaphore. We thought about the potential to add a attribute to the `thread` struct, but we determined that that didn't make sense, as it doesn't relate directly to a thread object.

# Task 2: Priority Scheduler
## Data structures and functions
```c
In thread.c/thread.h:
	struct thread
	{  
		...
		int donated_priority; /* Priority donated from it's aquired locks */
		struct lock *waits_for;  /* Lock thread waits for. */
		struct list acquired_locks;    /* Locks the thread currently holds. */
	};
	/* Will change to return max of donated_priority and priority. */
	int thread_get_priority(void) {}  

	/* Iterates through the ready list and returns the highest priority found. Used for immediate yielding. */
	int threads_get_max_priority(void) {}

	/* Modify to initialize `waits_for` to NULL and `donated_priority` to PRI_MIN. */
	static void init_thread(struct thread *t, const char *name, int priority) {}

In synch.c/synch.h:
	struct lock
	{
		...
		int donated_priority; /* Priority donated from a higher-priority thread waiting on this lock. */
		struct list_elem elem; /* Used to link the locks a thread currently holds. */
	};

	/* Will modify to initialise donated_priority to 'PRI_MIN' */
	void lock_init (struct lock *) {}

	/* Will modifiy for priority donation. Calls donate() */
	void lock_acquire(struct lock *lock) {}

	/* We will apply similar modifications as done with `lock_acquire`. */
	bool lock_try_acquire(struct lock *lock) {}

	/* Will modify to deal with immediate yields/higher-priority preference. */
	void lock_release(struct lock *lock) {}

	/* Handles priority donation. */
	int donate(struct lock *lock) {}
```
## Algorithms
1. Choosing the next thread to run:
 	1. To choose the next thread to run, we iterate through the ready list of threads. We choose the thread that has the
	highest effective priority.
2. Acquiring a Lock
 	1. When a thread tries to acquire a lock, we first test to see that the lock is free. If so, then we give the current
	thread the lock (and add it to the thread's `acquired_locks` list) and proceed normally. If not, and we are using the Priority Scheduler, then we call our donate
	method, which implements priority donation.
	2. `int donate(struct lock *lock) {}`
		1. Set the calling thread's `waits_for` to be the current lock.
		2. If the lock `donated_priority` is smaller than the calling thread's, then change the lock's `donated_priority` to the calling thread's effective priority. Also update the owning thread's `donated_priority` (max of it's existing `donated_priority` and the new one.
 	3. If the thread that owns the lock is blocked and waiting for another lock (i.e. `waits_for` is not NULL), we
		apply step 2 to the thread that owns the lock pointed to by `waits_for`. We repeat this process until we reach
		a thread that is either not blocked or not waiting on another lock.
		4. On success or failure, return an error/success code
3. Releasing a Lock  
	0. Remove the lock from the holding thread's `acquired_locks` list. 	
	1. If we are using the priority scheduler, recalculate the holding thread's `donated_priority` to be max of the `donated_priority` of the locks it holds (excluding the current lock). If it holds no locks then set it to `PRI_MIN`.
	2. Iterate through lock's `waiters` and find the thread with the highest effective priority rating.
	3. Give this highest-priority thread the lock (which will place it in the ready queue), and set it's `waits_for` to be NULL.
	3. Reset the lock's `donated_priority` to 'PRI_MIN'.
	4. If the previous holder's updated priority is less than `threads_get_max_priority()`, then
	yield the previous holder's thread.
4. Computing the effective priority
 	1. We return the maximum of the thread's `priority` and it's `donated_priority`.
5. Priority scheduling for semaphores and locks, and condition variables
 	1. Whenever a semaphore, lock, or condition variable plans to release a thread, we will iterate over the threads in the waiting list to find the thread with the maximum effective priority.
	2. Unblock the thread with highest effective priority.
6. Changing thread's priority
 	1. Set the thread's priority to the input.
	2. If the current thread's priority is no longer the highest effective priority, then yield the current thread.
7. Finding the maximum priority (`int threads_get_max_priority(void) {}`)
	1. Iterate through the ready list and return the highest effective priority.  

## Synchronization
Shared data:
* `donated_priority`: Because multiple locks may wait on the same lock, multiple locks may concurrently try to update the `donated_priority` variable of threads. In order to avoid race conditions we have a lock `donated_priority_lock` on the donated_priority. In order to read or write to that variable all functions we implement must first acquire the lock. This could potentially generate deadlocks but the specifications stated that we do not need to implement deadlock avoidance or recovery.
* `priority` and `waits_for: because these variables can only be set by the thread itself there's is no risk of race condition
* Lists: as stated in the previous part, whenever we will be manipulating list we will disable interupts because lists are not thread-safe.
* We assume that locks have been implemented thread safe.

## Rationale
For our scheduler and semaphores/locks/conditionals, we chose to iterate over the threads, instead of keeping a sorted list. This was because the priority ratings of the threads in each list could be while already in the list. We would therefore have to be constantly updating a list whenever a thread's priority in that list is changed. At the expense of a little runtime, we chose instead to avoid this problem. For nested donation and priority donation, we chose to create a helper method so that our changes to `lock_acquire` could be toggled off when our Priority Scheduler is not being run. To avoid overflowing the kernel stack for a thread, we chose to do nested donation iteratively instead of recursively.  For our overall design, one thing we considered was keeping a global table that would keep pointers to each of the locks, and then dealing with the necessary locks and their associated threads as needed when a lock is acquired or released. While such a design would provide a nice abstraction layer, allowing us to keep the thread struct unaltered, it would require more overhead control and would require a lot more code and time. We chose to use this design because it used less additional variables and was easier to understand and will be easier to program.

NOTE: The thread variable `int donated priority`, is not absolutely necessary but increases efficiency in that this value doesn't have to be recomputed every time, but only when one of the locks it owns receives a donation or is released.

# Task 3: Multi-level Feedback Queue Scheduler (MLFQS)
## Data structure and functions
```C
/****** New features or modifications in thread.h ******/
/*Add members to struct thread*/
struct thread
{
  ...
  struct fixed_point_t recent_cpu;
  int nice;
  /*list_elem for blocked_list*/
  struct list_elem blocked_elem;
}

/****** New features or modifications in thread.c ******/
/*An array for 64 ready queues with priorities from 0 to 63*/
struct list advanced_ready_queues[64];

/*A list of blocked threads*/
struct list blocked_list;

/*Number of ready threads*/
static long long num_ready_threads;

/*Initialize recent_cpu and nice when creating new thread*/
tid_t thread_create (const char *name, int priority,
               thread_func *function, void *aux)
{
  ...
  t->recent_cpu = 0;
  t->nice = 0;
}

/*The latest load average*/
struct fixed_point_t latest_load_avg;

/*Update recent_cpu for all threads(running, ready, blocked)*/
void update_recent_cpus (void);

/*Increase the recent_cpu of t by one*/
void update_recent_cpu (struct thread *t);

/*Update the global variable load_avg*/
void update_load_avg (void);

/*Recalculate priority for all threads, and update the ready queues accordingly*/
void update_priorities (void);

/*Update the priority of the thread t*/
void update_priority(struct thread *t)

/*Push a thead into advanced_ready_queues, only called by thread_unblock()*/
int push_to_ready (struct thread *t);

/*Remove a thread from advanced_ready_queues, only called by schedule()*/
int remove_from_ready (struct thread *t);

/*Update the queues and bookkeeping variables when blocking threads*/
void thread_block (struct thread *t)
{
  ...
  list_push_back (&blocked_list, &t->elem);
  --num_ready_threads;
}

/*Update the queues and bookkeeping variables when unblocking threads*/
void thread_unblock (struct thread *t)
{
  ...
  //list_push_back (&ready_list, &t->elem);
  push_to_ready(&t);
  list_remove(&block_list, &t->elem)
  ...
  ++num_ready_threads;
}

/*Do update at specific time step*/
void thread_tick (void)
{
  ...
  update_recent_cpu(t);
  if (timer_ticks() % TIMER_FREQ == 0){
    update_load_avg();
    update_recent_cpus();
  }
  if (timer_ticks() % 4 == 0) update_priorities();
}

/*Modify it according to the schedule algorithm mentioned below*/
static struct thread *next_thread_to_run (void);
```
## Algorithms
1. Choosing the next thread to run

 The following procedure must be atomic. If `num_ready_threads` equals 0, we shall run the idle thread;

 Otherwise, we go through each of the 64 queues from highest priority(63) to lowest priority(0) looking for a ready thread from the front of the queue. Suppose the first ready thread we find is t<sub>0</sub> whose priority value is p<sub>0</sub>. In this case,
 1. If t<sub>0</sub> has a lower priority than the current running thread t<sub>c</sub>, we continue running t<sub>c</sub>.
 2. If t<sub>0</sub> has a higher or equal priority than the current running thread t<sub>c</sub>, t<sub>0</sub> is chosen to run (and t<sub>c</sub> is sent to the back of the queue (Round Robin Rule)). If the t<sub>c</sub> that is being replaced with t<sub>0</sub> is blocked or dying, we do not send it back to the ready queue.

 Using push and pop operations while selecting the next thread and replacing the current thread, allows us to find the right thread with the highest priority while adhering to the Round Robin Rule. Using ready queues allows us to choose only ready threads. Blocked threads are maintained in a separate list.

2. Update priority queues

 The following procedure must be atomic.
 1. Every tick we update the `recent_cpu` for the current thread.
 2. Every second we update the `recent_cpu` and the `load_average` according to the specification.
 3. Every 4 ticks we update the priorities (the above two operations occur first if they coincide).
 4. We start the update operation by recording the number of elements in each of the 64 priority queues. For each queue, we then only update the first few elements as dictated by the recorded number. This is done because the update operation may change the thread priorities and thus might have to be moved to another queue. Since the moved thread are pushed to the back of the queue, computing the priority of only the first few threads allows us to avoid recomputation of priorities for the moved threads.
 5. After we compute the priority for each thread, we move it to the back of another queue according to the new priority, only if the priority of the thread changed.
 6. We then update the priority for the blocked threads.
 7. We also update the priority for the current thread. (If the new priority is lower than the highes priority, it yields to the scheduler).

3. Unblock a thread

 When a thread is waken up by current thread for whatever reason, we simply remove it from the blocked queue and push into ready queue according to its latest priority value. This priority value was updated last time `update_priorities()` is called, so it is consistent with the priority of all other threads.

 The caller of `thread_unblock()` should yield if any of the threads that are waken up has higher priority than the current thread.

4. Change of niceness

 The nice value is stored in TCB of each thread, and can be modified by the thread itself. But this modification can affect the priority only after the next time the priority is updated.

## Synchronization
We don't use any dynamic memory in this part. We have three potential public resources: `advanced_ready_queues`, `blocked_list` and `num_ready_threads`. They can only be accessed by `update_priorities()`, `thread_block()` and `thread_unblock()`. We disable interrupt for all these functions, so there are no synchronization issues.

## Rationale
By using an array instead of a list to store the 64 queues, we can avoid creating new `list_elem`, and can be easier to locate a specific queue by the index. The index of a queue provides information for its priority. The priority field in the thread struct can thus be used to find its corresponding ready queue. Also, using doubly linked lists allow us to access the first element and to add an element to the end in constant time. This is especially useful with our push/pop algorithm for threads when we need to identify the next thread to run while following the round robin rule. We maintain the number of ready threads in a separate constant which allows us to easily recompute the latest_load_avg and to make the decision of running idle threads.

The priority updating procedure is in-place, it only needs 64 extra space to record the size of each queue. And we don't need special flags in the thread struct to check if it is updated. We discussed an alternative approach of maintaining a separate boolean variable `updatedToggle` within each thread and then maintaining a global variable `currentToggleValue`, and then alternatively compare these values to decide if a thread was updated. For e.g. initially let all the values be false. As we go on updating priorities for the threads we toggle the thread level toggle values. We only update the threads where the value is not same as the global value. Once all threads are updated, we toggle the global value. Now next time we have to update the priorities we check do the same thing, but with true values as reference. This continues. We decided against it because we were not sure about how many threads would be in the queues and therefore, just having an array of 64 bytes is much more memory efficient.

## Additional Questions
1. Question 1
	1. Main:
		1. Create lock `l`
		2. Create sema `s` with state 0
		3. Create thread `A` with priority `PRI_DEFAULT + 10`, with args `s` and `l`
		4. Create thread `C` with priority `PRI_DEFAULT + 20`, with arg `l`
		5. Create thread `B` with priority `PRI_DEFAULT + 15`, with arg `s`
		6. sema_up(s)
		7. sema_up(s)
	1. Thread A(lock `l`, sema `s`):
		1. Acquires lock `l`
		2. Downs sema `s`
		3. Print message "Thread A Finished"
		4. Release lock `l`
	2. Thread B(sema `s`):
		1. Downs sema
		2. Print message "Thread B Finished"
	3. Thread C(lock `l`):
		1. Acquires lock `l`
		2. Print message "Thread C Finished"
		3. Release lock `l`

Assuming that there is a bug when priority donations are taken into account, then this test will not produce the correct
results.Main will create a lock `l` and a sema `s`, and then will create a thread named `A` that has priority `PRI_DEFAULT
+10`, with `s` and `l` as arguments. Then thread `A` will begin executing. `A` will acquire `l`, and then down `s`. This will
block thread `A`, returning execution to the main thread. From here (line 4 of main), a new thread `C` will be created, being
passed `l` as its argument. Now `C` will try to acquire `l`, but `l` is already owned by `A`. Because `C`'s priority is larger
than `A`'s, this will trigger a priority donation, temporarily raising `A`'s effective priority to `PRI_DEFAULT + 20`. From
this point `C` must wait on `A` to release `l` before it can proceed, so thread `C` gets blocked. The main thread resumes
execution at its line 5, which creates a thread `B` with priority `PRI_DEFAULT + 15`, with arg `s`. `B` will begin executing
and down `s`, which will block its execution. The only unblocked thread is main, so main then begins again starting at line 6.
It will  then up `s`, triggering `s` to wake one of its threads up. If there is a bug, then the output here should be wrong
because `A` has had a temporary priority donation from `C`, so its effective priority rating is higher than `B`'s. However,
the scheduler incorrectly chooses thread `B` to proceed next because `A`'s original priority is less than that of `B`. However, this is because it did not take into account its effective priority but rather its real priority. If it chose the next thread based on effective priority, then after the first sema_down `A` should execute prior to `B`, and not vice versa.

1. Actual Results:
	1. "Thread B Finished"
	2. "Thread A Finished"
	3. "Thread C Finished"
2. Expected Results:
	1. "Thread A Finished"
	2. "Thread C Finished"
	3. "Thread B Finished"


Question 2: The following scheduling decision uses the MLFQS scheduler and assumes that the `recent_cpu` is set at 0 at tick 0
for all the threads.

timer ticks | R(A) | R(B) | R(C) | P(A) | P(B) | P(C) | thread to run
 ------------|------|------|------|------|------|------|--------------
  0          |  0   |  0   |  0   |  63  |  61  |  59  |      A       
  4          |  4   |  0   |  0   |  62  |  61  |  59  |      A       
  8          |  8   |  0   |  0   |  61  |  61  |  59  |      B
 12          |  8   |  4   |  0   |  61  |  60  |  59  |      A
 16          |  12  |  4   |  0   |  60  |  60  |  59  |      B
 20          |  12  |  8   |  0   |  60  |  59  |  59  |      A
 24          |  16  |  8   |  0   |  59  |  59  |  59  |      C
 28          |  16  |  8   |  4   |  59  |  59  |  58  |      B
 32          |  16  |  12  |  4   |  59  |  58  |  58  |      A
 36          |  20  |  12  |  4   |  58  |  58  |  58  |      C

Question 3. The specification does not specify the order of different updates that need to happen in the scheduler. The
calculation of `recent_cpu` at every tick/every 4 ticks, the calculation of `priority`, and the calculation of `load_avg` can
assume any order. In the calculation shown above, this happens in the order calculate `recent_cpu` every sec for current
thread -> calculate `load_avg` -> calculate `recent_cpu` -> calculate `priority`.
