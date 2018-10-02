Design Document for Project 3: File System
==========================================

## Group Members

* Hadrien Renold
* Mark Sun
* Keshav Potluri
* Yichen Ruan 


# Task 1: Buffer Cache
## Data structures and functions
In file [/pintos/src/filesys/cache.c](/pintos/src/filesys/cache.c)

```c
struct list cache;

typedef struct cache_entry
{
	/* Used from outside the cache_entry, to navigate the list of cache_entries. */
	struct list_elem elem;		/* Used to create the list of cached entries. */
	block_sector_t sector;		/* Sector that the current cache entry represents. Doesn't necessarily
					 * correspond to the data that is currently stored.  */

	/* Data and data related metadata. */
	char data[BLOCK_SECTOR_SIZE];	/* Stored cached data. */
	block_sector_t actual_sector;	/* The actual sector number of the data. */
	bool dirty;

	/* Used for internal use for queuing operations on the cache entry. */
	int num_operations;			/* The number of operations currently queued. */
	int num_replacements; 			/* The number of replacemenets queued. */
	bool in_use;				/* Wether there is an operation currently running on the data. */
	struct condition data_replaced;		/* Wether the data has been replaced with new data (and the
						 * actual_sector number has changed). */
	struct condition all_actual_ops_finished; /* Wethere all operation on the current data have finished.
						   * Used for replacements. */
	struct condition current_op_finished;	/* Wether the current operation has finished. Acts as a queue
						* for the cache_entry read and write operations. */
	struct lock cache_entry_lock;		/* Lock on the cache_entry. Aquired before editing any shared
						 * values of the cache entry (including all the condition
						 * variables). */
} cache_entry_t;
```

In file [/pintos/src/filesys/cache.c](/pintos/src/filesys/cache.c) and [/pintos/src/filesys/cache.h](/pintos/src/filesys/cache.h):

```c
/* Initializes buffer with 64 empty entries on startup. */
void cache_init ();

/* Writes all the dirty entries in buffer and frees the memory on shutdown. */
void cache_destroy ();

/* Gets a block from cache corresponding to the input SECTOR. Performs cache
eviction and replacement if required. */
void cache_read_at (block_sector_t sector, void *buffer, off_t size, off_t offset);

/* Sets BUFFER into the cache. Performs cache eviction and replacement if
required. */
void cache_write_at (block_sector_t sector, void *buffer, off_t size, off_t offset);
```

In file [/pintos/src/filesys/cache.c](/pintos/src/filesys/cache.c):

```c
/* Writes all the dirty pages to disk. */
void write_cache_to_disk();

/* Writes a specific CACHE_ENTRY to disk. */
void write_entry_to_disk(cache_entry_t * cache_entry);

/* Reads a block of memory referred by SECTOR to a specific CACHE_ENTRY */
void read_sector_to_cache(block_sector_t sector, cache_entry_t *cache_entry);

/* Returns an entry pointed by SECTOR from cache. Returns NULL if not found. */
cache_entry_t * get_cached_entry(block_sector_t sector);

/* Gets the next entry to evict based on our replacement policy. */
cache_entry_t * get_entry_to_evict();

```
## Algorithms
We implement an LRU replacement policy by maintaining the `cache` list ordered with the most recently used in the front of the list. Every time a request is made for a cache entry we move the given entry to the front of the list.

The overall pattern for each cache entry is to have three list:
1. A list of waiters on the condition variable `current_op_finished` which are waiting for the current cache entry to be free to access (read/write) it
2. a list of waiters on the `all_actual_ops_finished` which are waiting on the previous list to be empty in order to replace the cache entry (ie evict it and bring in a new entry)
3. a list of waiters on the `data_replaced` condition variable which are waiting for the entry to be replaced to access the future entry.

### Reading/Writing from cache
To access from the cache we first determine if an element is in the cache as implemented by the method `get_cached_entry`. To do this we iterate through the list and check if the sector we are looking for is in the list.
- If the entry is in the cache:
  - if the `actual_sector` is the one we desire
    - if the entry is not `in_use` we do our operation
    - otherwise we call wait on the condition variable `current_op_finished`. The thread will get woken up when the previous thread has finished accessed the cached entry, effectively serializing all requests to the specific cache entry.
  - otherwise we wait on the condition variable `data_replaced` and will thus get woken up when the cache entry has been filled with a new block. We then check wether the `actual_sector` is the one we are trying to read from or not. If it is we wait on the cache entry to be free. Otherwise we wait on `replaced` again (this means we are waiting for a future replacement as multiple replacements can be queued).
- If the entry is not in the cache we call `get_entry_to_evict` to get the cache entry to evict as per our replacement policy. As we implement LRU this is the last entry in the list. To evict a entry we do the following:
  - we acquire the `cache_entry_lock`
  - we set the sector number to the sector number of the future block so that we can start queueing requests for those already and not receive anymore requests for the previous sector
  - we wait on `all_actual_ops_finished` to make sure that all pending accesses to the block are completed
  - we write the block to disk if the entry is dirty
  - we reset the dirty bit
  - we load the block into the cache entry
  - we broadcast on `data_replaced` to wake up all threads waiting to access the new block
  - we release the `cache_entry_lock`

The next time we are scheduled we will have access to a cached entry of the block we are trying to access and hence can do our read/write. If we write we also take care of setting the dirty bit.

To finalize the read/write operation we:
- update `in_use` and signal on `current_op_finished` so as to wake up the next thread waiting to access the cache
- signal on `all_actual_ops_finished` if `num_ops` is zero. This way if we are waiting to evict the page we can now proceed to evict it.

## Synchronization
We decided to replicate the signatures of `inode_read_at` and `inode_write_at` in our cache, so that all the handling of lock and synchronization can be done from within the cache and is transparent to the caller.
In order to avoid any concurrent access to the cache entry we have a lock per cache entry which we acquire for the condition variables or when reading or writing data to the cache entry. When reading metadata we do not acquire the lock otherwise this would amount to a global cache lock; instead we read the data and before doing any operation on the data, at which point we have the lock, we check that the metadata is still what we expected it to be and if it isn't we start over. This should be rare and thus not decrease performance too much.

We use condition variables so that we realease the lock on the wait so that no deadlock is created.
To avoid concurrent modifications to the `cache` list we disable interupts on read or edits to that list.

## Rationale
We approached this task in multiple different ways during our design process. One of our approach was to implement a `lock` on `inode` as well as to implement another `lock` for the `cache_entry`. The way it would work was that a process would acquire a lock on an `inode`, then request a read or write on the required sectors to the cache by acquiring lock on individual `cache_entry` elements. Each time when the entry is accessed, we put it to the front of the cache list. If a process demands a sector from an `inode` that is not present in the cache, we mark the last cache entry for eviction. If there is an operation being performed on the entry, our eviction waits till the operation is complete. Once done, the new sector is fetched into the entry (after write back if dirty) and pushed to the front of the list. This design follows LRU policy and provides synchronization for different files and directories, but fails for different sectors.

Another approach we thought of was to use semaphores to perform reads and writes to and from the cache. In `cache_entry` we maintain different count of number of threads reading the entry, a flag for whether data is being written to the entry and a flag to identify if the entry is performing disk I/O. We maintain a semaphore on the entry and a lock. For reads we would check if there is a write or a disk I/O being performed. If yes, we do a sema down and make the process wait. If not, we continue with the read. For writes and disk I/Os, in addition to the above two conditions, we also check if the number of files that are currently reading the entry is greater than 0. If yes, we do a sema down and make the process wait. If not we continue with the process and once done, do a sema up to wake the waiting processes. This was a really good approach that would have allowed us asynchronous operations on all levels as well as multiple asynchronous reads. We however ran into problems of activating the correct process without using considerable additional memory, because of the order in which the sleeping threads wake up is not consistent. We also were not sure about how to handle edge cases when the same sectors receive continuous read requests, thus blocking other operations.

We tried a number of other approaches and ran into problems with either synchronization or eviction during edge cases. We finally finalized our design which allows us to queue the requests for all the `cache_entry` elements. The requests are queued per sector and therefore, reads and writes on the same sector are serialized, but otherwise any other operations can be handled asynchronously. We queue an eviction request which waits for all current requests on the sector to finish before evicting the entry. We then also queue any future requests for the sectors that are going to be fetched, to the appropriate `cache_entry` and therefore avoid multiple fetches for a sector that is already scheduled to be fetched. Finally, with each access, we also make sure to push the corresponding `cache_entry` to the front of the cache list to ensure an LRU policy.

# Task 2: Extensible Files
### Data structures and functions
In file [/pintos/src/userprog/syscall.c](/pintos/src/lib/user/syscall.c):
```c
/*** ADDITIONS ***/
static void syscall_handler (struct intr_frame *f){
	...
    case SYS_INUMBER:
    /* Will perform argument validation and call syscall_inumber()*/
    	...   
}

/* Uses inode_get_inumber() */
int syscall_inumber(int fd);
```
In file [/pintos/src/filesys/inode.c](/pintos/src/filesys/inode.c):
```c
/*** ADDITIONS ***/
#define INODE_NUM_DIRECT 12
#define INODE_NUM_INDIRECT 1
#define INODE_NUM_DOUBLY_INDIRECT 1
#define INODE_NUM_SECTORS INODE_NUM_DIRECT + INODE_NUM_INDIRECT + INODE_NUM_DOUBLY_INDIRECT
#define INODE_DIRECT_SIZE NUM_DIRECT * BLOCK_SECTOR_SIZE
#define INODE_INDIRECT_SIZE NUM_INDIRECT *(BLOCK_SECTOR_SIZE/4) * BLOCK_SECTOR_SIZE

#define INODE_INDIRECT_OFFSET INODE_DIRECT_SIZE
#define INODE_DOUBLY_INDIRECT_OFFSET INODE_DIRECT_SIZE + INODE_INDIRECT_SIZE

  /* Iterates through the direct blocks, the indirect block, and the doubly
     indirect block, and tries to free each used sector. Returns true if the
     the inode's sector are successfully freed, or false if an error occurs. */
  bool inode_free_sectors (struct inode *inode);

  /* Extends the INODE's size by allocating NUM_SECTORS new sectors.
     Returns true if the inode was extended, or false if not enough
     memory is available or the free_map file could not be written to. */
  bool inode_extend (struct inode *inode, size_t num_sectors);


/*** MODIFICATIONS ***/

/* On-disk inode.
   Must be exactly BLOCK_SECTOR_SIZE bytes long. */
struct inode_disk
  {
    /* Direct sectors. */
    block_sector_t sectors[INODE_NUM_SECTORS];
    off_t length;                       /* File size in bytes. */
    unsigned magic;                     /* Magic number. */
    uint32_t unused[111];               /* Not used. */
  };

/* In-memory inode. */
struct inode
  {
    ...
    // struct inode_disk data;    // We will remove this
  };

  /* Explained below. */
  bool inode_create (block_sector_t sector, off_t length);

  /* Explained below. */
  static block_sector_t byte_to_sector (const struct inode *inode, off_t pos);

  /* Explained below. */
  off_t inode_read_at (struct inode *inode, void *buffer_, off_t size, off_t offset);

  /* Explained below. */
  off_t inode_write_at (struct inode *inode, const void *buffer_, off_t size, off_t offset);

  /* Explained below. */
  void inode_close (struct inode *inode);
```

In file [/pintos/src/filesys/free-map.c](/pintos/src/filesys/free-map.c):
```c
/*** ADDITIONS ***/
/* Used for functions which operate on the free_map */
static struct lock free_map_lock;

/*** MODIFICATIONS ***/
/* Allocates CNT sectors from the free map and stores
   them into *SECTORS.
   Returns true if successful, false if not enough
   sectors were available or if the free_map file could not be
   written. */
bool
free_map_allocate (size_t cnt, block_sector_t *sectors)
```

### Algorithms
##### Syscall: `int inumber(int fd);`
Kernel will validate arguments, and then return result of `inode_get_inumber(inode)`, with `inode` being the inode associated with the fd that is passed in, to the user. Because the syscall already exists in the user library, we will simply add another case in the syscall_handler to handle the `SYS_INUMBER` case.
##### Reading a file/`byte_to_sector()`:
Currently these functions rely on the data blocks all being contiguous and the on disk inode being in the `struct inode`, which we are changing. Instead of accessing the `inode_disk` from `struct inode`, we will retrieve the `inode_disk` from the sector number of the inode. Also, we will need to modify the existing functions to handle our updated on disk inode structure. Each of these functions will now have to iterate through the sectors on the on disk inode, first through the direct blocks, then the indirect blocks, and then the doubly indirect blocks.
##### Creating/Extending a file:
Currently the data blocks are all allocated contiguously or not, as `free_map_allocate()` only tries to find contiguous blocks in the bitmap to use. This leaves us vulnerable to external fragmentation, so to avoid this we change `free_map_allocate (size_t cnt, block_sector_t *sectors)`. We change it so that it tries to find `CNT` available blocks to use, which need not be contiguous, and if enough are available, it stores the sectors into `SECTORS`. We will need to change `inode_create()` so that it uses the updated function. Furthermore, when writing past EOF, we will need to allocate new blocks if they are available, so we will need to modify `inode_write_at()` so that it calls `inode_extend()` in this case.
##### Closing an inode:
Upon closing the last reference to the inode, we will call `inode_free_sectors()` to iterate through all the sectors (direct, indirect, and doubly indirect) and and release its blocks.

### Synchronization
We use a lock on the free_map to prevent race conditions when allocating new blocks or releasing them. This way only one inode can be extended at a time, so we avoid having different inodes being mapped to the same data blocks. Functions operating on the same sector are serialized (explained in next part).
### Rationale
We decided to model our sector blocks like that of Unix, which has 12 direct sectors, 1 indirect sector, 1 doubly indirect sector, and 1 triply indirect sector. However, since the maximum partition size is 8 MB, we did not include the triply indirect sector. Our on disk inode can support files up to (12 * 512) + (1 * (512/4) * 512) + (1 * (512/4) * (512/4) * 512) = 8460288 bytes, which is sufficient.

# Task 3: Subdirectories
## Data structures and functions
In file [/pintos/src/filesys/directory.c](/pintos/src/filesys/directory.c):
```C
struct dir_entry
{
  ...
  bool is_dir;                    /* Is directory or file? */
}

/* Check if ENTRY1 and ENTRY2 are the same */
bool same_dir_entry (struct *dir_entry entry1, struct *dir_entry entry2)

/* Locate the dir_entry specified by NAME, return the lowest dir_entry, or return NULL on failure. Support both absolute and relative path. */
struct dir_entry *locate_entry (char *name);
```

In file [/pintos/src/filesys/filesys.c](/pintos/src/filesys/filesys.c):
```C
/* Modify the following functions to support subdirectories */
bool filesys_create (const char *name, off_t initial_size);
struct file *filesys_open (const char *name);
bool filesys_remove (const char *name);
```

In file [/pintos/src/filesys/file.h](/pintos/src/filesys/file.h):
```C
/* Data structure to track files and their fds */
struct open_file
{
  int fd;                    /* File descriptor */
  struct file *_file;        /* file struct in file.c */
  struct list_elem ofelem;   /* list_elem to be added to a thread's open_file_list */
  bool is_dire;              /* Is directory or file? */
}
```

In file [/pintos/src/threads/thread.h](/pintos/src/threads/thread.h):
```C
struct thread
{
  ...
  struct dir_entry cwd;      /* Current working directory */
}
```

In file [/pintos/src/userprog/syscall.c](/pintos/src/userprog/syscall.c)
```C
/* Implement the corresponding syscalls */
bool syscall_chdir (const char *dir);
bool syscall_mkdir (const char *dir);
bool syscall_readdir (int fd, char *name);
bool syscall_isdir (int fd);
int syscall_inumber (int fd);
/* Update existing syscalls for fds corresponding to directories */
int syscall_open (const char *file);
int syscall_read (int fd, void *buffer, unsigned size);
int syscall_write (int fd, void *buffer, unsigned size);
bool syscall_remove (const char *file);
tid_t process_execute (const char *file_name)
```

In file [/pintos/src/filesys/inode.c](/pintos/src/filesys/inode.c)
```C
struct inode
{
  ...
  struct lock mutex;        /* Mutex lock for each inode */
}
/* Update the following functions to guarantee synchronization */
off_t inode_read_at (struct inode *inode, void *buffer_, off_t size, off_t offset);
void inode_close (struct inode *inode);
off_t inode_write_at (struct inode *inode, const void *buffer_, off_t size, off_t offset);
```
## Algorithms
### Locate a file/directory:
The first two `dir_entry` of a `dir` is set to be `.` and `..` by default, which point to the sector number of `dir` itself or its parent directory respectively (for root directory, they all point to the root itself). That also means for the `dir_entry` of a file, our read/write shall start from `2 * sizeof dir_entry`.

If it is an absolute path, we start from the root directory, following the `inode_sector` field of the `dir_entry` struct to search for the next subdirectory, until we reach the destination or encounter an error; If it is a relative path, we start from the `cwd` of the current thread, following the `.` or `..` entries to find the target file/directory. The `is_dir` field of the `dir_entry` can help us identify whether there is a subdirectory at the next level.

### Compare directory entries:
To check if two `dir_entry`s are the same, we check that they have the same name, and are under the same parent directory. We can compare the parent directory by fetching the second entry (`..`) from `dir_entry`'s inode, then compare the sector number. Since a directory will never have two entries with the same name, we don't need to check what is pointed to by the entry.

### Remove a file/directory:
If it is a file, we first find the desired file/directory in the file system, then remove it with `dir_remove ()`.

We don't allow deletion of a directory that is open by a process or in use as a process’s current working directory. So if it is a directory, we also need to check all its slots are unused (except `.` and `..`), and go over the thread list to make sure it is not the current working directory of any processes using `same_dir_entry ()`. We can actually remove the file/directory only after passing all those checks.

### Create a new directory:
We first check the path is valid, then find a free slot of the lowest directory, replace it with the target `dir_entry`. Finally, we set the first two entries of this new directory to be `.` and `..`.

### Read from an open directory:
Similar to read from an open file. We follow the `fd` to find the `open_file` struct, use the `is_dir` field to check if it is a directory. If yes, use the `_file` field to get its `file`, then read the names of the used `dir_entry` from it (except `.` and `..`).
## Synchronization
We have a mutex lock for each `inode`. We will acquire the lock before doing any low level disk IO. Since all open `inode`s will be inserted to the `open_inodes` list no matter it is a file or a directory, this strategy allows us to serialize all the file system operations on the same sectors.
## Rationale
We decide to store the parent pointer of each directory not only in memory but also in the disk. This just costs a little bit extra space, but it makes the memory management much easier. Otherwise we will have to keep all the intermediate directories in memory, which could further cause lots of synchronization issues. Besides, saving `.` and `..` as directory entries is going to facilitate the implementation of relative path interpretation and directory lookup.
# Design Document Additional Questions

## Question 1
##### Write-behind
We implement the write behind as follows. In the cache, for each `cache_entry`, we maintain a `dirty` flag. Whenever an entry is written to, we set the flag to true. At the system startup, we spawn a thread that writes the dirty entries to the disk. This thread sleeps in a non-blocking way, as implemented in the earlier projects. Every few seconds (maybe 5), the thread wakes up, loops through the cache to identify the dirty entries, writes them to the disk, sets the corresponding `dirty` flag to false and then sleeps back.


##### Read-ahead
We would have implemented the read ahead as follows. A read or a write first identifies the appropriate sector required for the action based on the block pointers stored in an `inode`. If the corresponding sector is not found in a cache, a fetch is performed to bring the appropriate sector to the memory and the data is then served to the calling process. In order to do an asynchronous read ahead fetch, we spawn a separate thread. We pass the all the parameters like inode, offset, number of bytes to read etc. which we already have in the initial fetch, to this new thread. This thread calculates the number of more sectors to be fetched from the input, and identifies if they are already present in the cache. If not, the thread identifies the locations for the next sectors from the inode and fetches the next sectors. The number of sectors to be pre fetched can be set to a pre determined limit.
