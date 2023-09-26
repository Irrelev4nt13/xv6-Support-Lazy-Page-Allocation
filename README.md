# xv6-Support-Lazy-Page-Allocation
📄🖥️Implementation of lazy page allocation for xv6 for the operating systems course

## Contributing 
Michail Chatzispyrou

## Compile & Run
To start the emulator we run `make qemu`.<br/>Then we can write any command supported by the corresponding shell.<br/>Finally, we run `make grade` if we want it to run all the tests.<br/>

## Modeling & Comments
### Step 1 – Print the page table
```c
void vmprint_rec(pagetable_t pagetable, int page_directory) {
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0) {
      for (int j = 0; j < page_directory; j++)
        printf(".. ");
      uint64 child = PTE2PA(pte);
      printf("..%d: pte %p pa %p\n", i, pte, child);
      vmprint_rec((pagetable_t)child, page_directory + 1);
    }
    else if (pte & PTE_V) {
      uint64 child = PTE2PA(pte);
      for (int j = 0; j < page_directory; j++)
        printf(".. ");
      printf("..%d: pte %p pa %p\n", i, pte, child);
    }
  }
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  int page_directory = 0;
  vmprint_rec(pagetable, page_directory);
}  
```
For this particular step I used all the necessary hints given in the pronunciation. In more detail I traverse the page recursively as mentioned in `freewalk`. I have made a secondary recursive function which is called by `vmprint()`. If the `pte` "shows" at a lower level then we print the corresponding `..` to show that we are changing the level. Then we print the requested data for `pte` and continue the recursive call increasing the level. If `pte` is only valid then we have reached the last level so we print the corresponding `..` for switching levels and the necessary elements of pte without calling the recursive function.

### Step 2 – Eliminate allocation from sbrk()
The new `sbrk` can be written as: 
```c 
uint64
sys_sbrk(void)
{
  uint64 addr;
  int n;
  
  addr = myproc()->sz;

  argint(0, &n);
  
  myproc()->sz += n;
  
  return addr;
} 
```
More specifically, with the command `addr = myproc()->sz;` we store the old size of the process which we want to return. Then, with the command `argint(0, &n);` we fetch the size `n` which will be added to the total size of the process. Finally, with the command `myproc()->sz += n` we increase the size of the process without allocating memory and then return its original size.<br/><br/>
Question:<br/>
> The “usertrap(): ...” message is from the user trap handler in the
kernel/trap.c file. It has caught an exception that it does not know how to handle.
Make sure you understand why this page fault occurs. The
“stval=0x0000000000004008” indicates that the virtual address that caused the page fault is 0x4008. 

Answer:<br/> 
> We notice that the value of scause is 15, which corresponds to page faults. The page fault we expected to occur since `sbrk` does not actually commit a page. Finally, the stval=0x00000000000005008 (the value on my machine was different from the output) corresponds to the virtual address that causes the page fault

### Step 3 – Lazy allocation
The new `sbrk` can be written as:
```c
uint64
sys_sbrk(void)
{
  uint64 addr;
  int n;

  addr = myproc()->sz;

  argint(0, &n);

  if(n < 0)
    myproc()->sz = uvmdealloc(myproc()->pagetable, addr, myproc()->sz + n);
  else
    myproc()->sz += n;

  return addr;
}
```
The `sbrk` has been suitably modified to handle negative arguments as well. More specifically after assigning the value to `n` we check if it is negative or not, in case it is not negative the command `myproc()->sz += n;` is executed in order to increase the size of the process as in Step 2. Otherwise, if `n` is negative then we deallocate the user pages to reduce the size of the process which we assign to `myproc()->sz` via `uvmdealloc(...)` which returns the new process size.<br/>

Changes had to be made to `usertrap(...)` as well.
```c
else if (r_scause() == 13 || r_scause() == 15) {
    char *mem;
    if (r_stval() >= p->sz || r_stval() < p->trapframe->sp)
      setkilled(p);
    else if ((mem = kalloc()) == 0)
      setkilled(p);
    else if (mem != 0) {
      memset(mem, 0, PGSIZE);
      if (mappages(p->pagetable, PGROUNDDOWN(r_stval()), PGSIZE, (uint64)mem, PTE_X | PTE_W | PTE_R | PTE_U) != 0) {
        kfree(mem);
        setkilled(p);
      }
    }
  }
```
In order to deal with the possibility that the virtual address is not translated to physical, the function must be modified to allocate the required page space and do the necessary mapping, but if the size is out of bounds then the process will be "killed" directly . The binding and mapping process is with the `uvmalloc()` function which I was prompted to use.

Changes had to be made to `walkaddr(...)` as well.
```c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  struct proc* p = myproc();

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0) {
    if (va < p->sz && va >= p->trapframe->sp) {
      char *mem;
      if ((mem = kalloc()) == 0)
        return 0;
      else {
        memset(mem, 0, PGSIZE);
        if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, PTE_X | PTE_W | PTE_R | PTE_U) != 0) {
          kfree(mem);
          return 0;
        }
      }
    } 
    else
      return 0;
  }
  pa = PTE2PA(*pte);
  return pa;
}
```
To be able to handle the case where a process passes a valid address from `sbrk()` to a system call like `read()` or `write()`, but the memory for this address has not yet been allocated so we need to modify the `walkaddr(...)` function. Specifically, after a system call the kernel will never go to `usertrap(...)` so the page fault must be handled in the `walkaddr(...)` function which is called by both `copyin(... )` as well as `copyout(...)` to transfer data from user to kernel and vice versa. The function previously checked if the `pte` exists or is valid or is accessible by the user so that 0 is returned since they were not allowed, from now on in lazy allocation everything is allowed so it is enough that one of the three conditions apply. Therefore, if the virtual address is within bounds then the binding and mapping process is performed otherwise 0 is returned indicating that no mapping was achieved. Finally, if the bind and map were executed successfully, the physical address is returned.

  
Additional changes were made to both the `uvmunmap(...)` and `uvmcopy(...)` functions. 
```c
Before:
   if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
After:
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
```
```c
Before:
   if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
After:
    if((pte = walk(old, i, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
```
Regarding the `uvmunmap(...)` changes we modified the function and "call" it to ignore pages that are either not created or not mapped yet. In addition, `uvmcopy(...)` is mainly used during the `fork()` process, so it must be modified to ignore possible pages that have either not been created or mapped yet during the parent-child copy. Finally, we notice that `uvmcopy(...)` calls the `uvmunmap(...)` function to stop matching the map in case of memory allocation failure, however, we have already modified the function to ignore the corresponding pages.
