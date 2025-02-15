# UNSW Extended Operating Systems 2018 #

# Implementing OS-161 File System #

## Required Data Structures ##

## File Descriptor Table (fdt) ##

- Each process has a pointer to a unique file descriptor table
structure that keeps a count of used fd entries, and an array of
pointers to OFT entries (struct fdt *p_fdt).

- We use an array of pointers to OFT entries as fd’s range from  0 to
OPEN_MAX (32). The fd can then be used as a direct index and the
array won't be too large.

- We keep a count of how many slots in the fdt have already been
allocated.

- Each fdt has its own mutex to prevent threaded concurrency issues
(i.e. one slot left in fdt, thread A passes check that there is room
in table, thread B then runs and passes same check, now two threads
want to acquire a slot in the table and there is only one).

struct fdt {
    int count;
    struct lock *fdt_mutex;
    struct oft_entry *fdt_entry[OPEN_MAX];
};



## Open File Table (oft) ##


- A structure that keeps track of files opened by user-level processes.

- As oft entries are pointed to by an index (fd) in an fdt and not
directly indexed themselves, they do not necessarily have to be part
of a table (especially not in an array due to the many thousands that
are open at any given time).

- Standalone structs are therefore ideal  as they can be easily
malloc’d, moved and destroyed without affecting an overall table
structure.

- There is a need to maintain a pointer to the underlying vnode that
represents the file, as well as some metadata (flags/permissions,
seek position, and a reference count).

- The oft entry also needs a mutex to ensure atomic io operations to
the file represented by it (as multiple fd’s can reference the same
oft entry and multiple oft entries can reference the same vnode).

struct oft_entry {
    struct lock *oft_mutex;
    struct vnode *vn;
    mode_t mode;        // kept for portability
    int flags;
    off_t seek_pos;
    int ref_cnt;
};



## Syscall Function Planning ##


# sys_open #

-Check dependencies/prerequisites and passed arguments are valid
(check filename not null or too long, the current processes fdt
exists and the current amount of fd’s currently in use isn’t at
limit. Return required error codes where needed).

- Also need to duplicate filename as vfs_open destroys passed in
strings. Furthermore, need to use copyinstr() if filename is a
user-level pointer because it could overflow userspace. Note that
each process preallocates fd 0, 1 and 2 to stdin,stdout and stderr
inside runprogram.c which is why some file names can come from kernel
space.

- Retrieve the vnode specified by path/filename with vfs_open then
create an oft entry containing the metadata, acquire the next
available fd and link it to the oft entry.

- Garbage collect (free, null-out etc.)



# sys _close #

- Dependency/argument checks (valid fd, fdt table exists, oft entry
exists).

- Acquire mutex locks on the current process fdt and oft entries whilst
modifying them to maintain atomicity between syscalls.

- If the oft entry is being pointed to by multiple fd entries, then do
not destroy/cleanup the oft entry, simply decrement its reference
count, and NULL out the the requested fd link.

- If the fd entry is the sole reference to the oft entry, then close
the vnode and cleanup/destroy the entry.



# sys_read / sys_write #

- Read and Write are essentially the same operation, just opposites in
terms of data flowing from a file to a buffer (read) or from a buffer
to a file (write). Therefore we’ll abstract them out as wrappers that
call the same underlying function with a write or read flag
(sys_io(common perms…, readwriteflag)).

- Dependency/argument checks (valid fd, fdt table exists, oft entry
exists, buffer is valid).

- Acquire a lock on the oft entry to prevent multi-threaded concurrency
issues between multiple syscalls (read, write, close, lseek) as they
can share the same oft entry (more specifically an offset into the
file).

- Check the request flag(s) in the oft entry and only proceed if
matching read/write operations - ie O_RDONLY etc.

- Initialise a uio_struct and pass it into VOP_WRITE or VOP_READ
depending on the readwriteflag. These macros end up calling a series
of functions that call copyin() for write or copyout() for read if
the segment of memory is in user space, thus handling the check on
buffer. They also perform the underlying transfer of data.

- Update the seek pos in the oft entry to match that of the vnode after
the io operation and return the number of bytes successfully
read/written.



# sys_dup2 #

- Dependency/argument checks (valid fd, fdt table exists, oft entry
exists).

- Acquire mutex locks on the current process fdt and oft entries whilst
modifying them to maintain atomicity between syscalls.

- As the resultant fd’s point to the same oft entry, any updates by one
fd (such as lseek), will be seen by the other fd. Note that if the
new fd already has its own oft entry, it will first be closed before
pointing to the old fd oft entry.



# sys_lseek #

- Set up local variables needed (64 bit local variable for 64 bit
offset, fstat struct for getting metadata to determine file size with
VOP_STAT()).

- Dependency and argument checks (fd is valid, fdt is valid)

- As the pos argument is a 64 bit value, and is the second argument, it
passed in as two 32 bit values in the a2,a3 registers, so join them
together using join32to64() and store in local 64 bit variable).

- Will have to use copyin() to obtain whence from the stack (registers
only accommodate 4 variables and 64 bit pos took up 2).

- Check current seek + pos is valid and if so calculate the new
position by adding to an offset specified in the file by a flag (i.e.
SEEK_END).

- As we are returning a 64 bit value we will have to split it with
split64to32() into the return registers v0 and v1, which means we
will also have to pass in the trap frame as an argument

- As the syscall handler (syscall.c) automatically sets v0 t0 retval we
will also set retval to be v0 (make it a nop and prevent overwriting
the return value crafted above).


# sys_getpid #

- The actual implementation of the sys_getpid function is trivial as it
can never fail and just returns the pid attached to the calling
processes struct. The acquiring of pid’s by processes is where the
bulk of the work happens.

- We assign pid’s to processes (upon process creation) incrementally
from PID_MIN until PID_MAX is hit, however, because processes can
exit and free up pid’s, we wrap around and search again to find one
of these free pid’s once PID_MAX is hit.

- We achieve this by having a system process table (which is just a
doubly-linked list of all the process structs) and a pointer to the
last process made. We can then simply start checking pid’s from
lastprocs location in the list (and start at its pid) and circulate
around the linked list until we get a free pid.

- We also use a proc_cnt global variable to ensure that once we have
PID_MAX pid’s allocated a new process cannot be made (it would just
stay in pid acquire loop forever until a process exits).


# sys_fork #

- We create the child process with proc_create_runprogram which
essentially sets up a struct with fields to be filled in i.e. empty
fdt table, null address space. It does however initialize some fields
as copies of the parent program (same current working directory and
name) and also acquires a unique pid.

- We then fill in the process struct taking into consideration that
some objects are shared between the parent and child instead of
copied.

- We copy the address space of the parent with as_copy so that the
child has the same text, variables etc. (it is essentially the same
process but in a different address space).

- The fdt table is process specific so we create an fdt for the child
process but the actual fdt entries they point to that represent open
files are not copied but instead shared. For this reason we do not
create any new oft entries but instead loop through the parent
processes fdt table and copy across any pointers to the newly created
child fdt (at the lowest available fd index by convention). We also
initialize the fdt count to that of the parent processes.

- Since fork creates a new process it needs to return twice on success
but only once on failure (because the new process will not be made in
this case). Furthermore, if successful the two processes must return
different values for identification (parent returns child pid, child
return 0) and so we must have two different trapframes.

- In the parent, the trapframe can remain where it is (kernel stack)
but for the child process it needs to copied onto the process stack
(required by mips_usermode). To achieve this we malloc a copy of the
parents trapframe, then once we are in the child's thread (called
enter_forked_process) we create a local trapframe struct (so it goes
on the stack) then we copy in the contents from the parent trap frame
on the heap via a pointer and then free it.

- Now we have two different trap frames so we can set the different
return values in each in the respective functions.

- We also ensure we have everything set up and valid before we call
thread_fork because it eventually calls enter_forked_process and then
mips_usermode from which you cannot return and so any errors must be
caught prior to this as it is assumed the fork is successful at this
stage.
