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
Για το συγκεκριμένο βήμα χρησιμοποιήσα όλα τα απαραίτητα hints που δώθηκαν στην εκφώνηση. Αναλυτικότερα διασχίζω την σελίδα αναδρομικά όπως αναφέρεται στην freewalk. Έχω φτιάξει μια δευτερεύουσα αναδρομική συνάρτηση η οποία καλείται από την vmprint(). Αν το pte «δίχνει» σε κατώτερο επίπεδο τότε τυπώνουμε τις ανάλογες `..` για να δείξουμε ότι αλλάζουμε επίπεδο. Μετά τυπώνουμε τα ζητούμενα στοιχεία για το pte και συνεχίζουμε την αναδρομική κλήση αυξάνοντας το επίπεδο. Αν το pte είναι μόνο valid τότε φτάσαμε στο τελευταίο επίπεδο οπότε τυπώνουμε τις ανάλογες `..` για την εναλλαγή επιπέδου και τα απαραίτητα στοιχεία του pte χωρίς την κλήση της αναδρομικής συνάρτησης.

### Step 2 – Eliminate allocation from sbrk()
Η νέα `sbrk` μπορεί να γραφτεί ως εξής: 
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
Αναλυτικότερα, με την εντολή `addr = myproc()->sz;` αποθηκεύουμε το παλιό μέγεθος της διεργασίας το οποίο θέλουμε και να επιστρέψουμε. Στη συνέχεια, με την εντολή `argint(0, &n);` «φέρνουμε» το μέγεθος n το οποίο θα προστεθεί στο συνολικό μέγεθος της διεργασίας. Τέλος, με την εντολή `myproc()->sz += n` αυξάνουμε το μέγεθος της διεργασίας χωρίς να δεσμεύσουμε μνήμη και έπειτα επιστρέφουμε το αρχικό της μέγεθος.<br/><br/> 
Ερώτημα:<br/>
> Το μήνυμα “usertrap(): ...” περιέχεται στο διαχειριστή user traps στο αρχείο
kernel/trap.c. Αντιμετωπίζει ένα exception που δεν γνωρίζει πώς να το χειριστεί. Θα
πρέπει να κατανοήσετε γιατί συμβαίνει αυτό το σφάλμα σελίδας. H ένδειξη
“stval=0x0000000000004008” υποδεικνύει ότι η εικονική διεύθυνση που προκάλεσε το σφάλμα σελίδας είναι η 0x4008. 

Απάντηση:<br/> 
> Παρατηρούμε ότι η τιμή του scause είναι 15, η οποία αντιστοιχεί σε σφάλματα σελίδας (page fault). Το σφάλμα σελίδας περιμέναμε να εμφανιστεί αφού η `sbrk` δεν δεσμεύει πραγματικά μια σελίδα. Τέλος, η ένδειξη stval=0x0000000000005008 (η τιμή που έβγαζε στο δικό μου μηχάνημα ήταν διαφορετική από της εκφώνησης) αντιστοιχεί στην εικονική διεύθυνση που προκάλεσαι το σφάλμα σελίδας. 

### Step 3 – Lazy allocation
Η νέα `sbrk` μπορεί να γραφτεί ως εξής:
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
Η `sbrk` έχει τροποποιηθεί κατάλληλα ώστε να διαχειρίζεται και αρνητικά ορίσματα. Πιο συγκεκριμένα μετά την ανάθεση της τιμής στο `n` ελέγχουμε αν είναι αρνητικό ή όχι, στην περίπτωση που δεν είναι αρνητικό εκτελείται η εντολή `myproc()->sz += n;` προκειμένου να αυξηθεί το μέγεθος της διεργασίας όπως και στο Βήμα 2. Διαφορετικά, αν το `n` είναι αρνητικό τότε θα αποδεσμεύσουμε τις σελίδες χρήστη ώστε να μικρύνουμε το μέγεθος της διεργασίας το οποίο και αναθέτουμε στο `myproc()->sz` μέσω της `uvmdealloc(...)` η οποία επιστρέφει το νέο μέγεθος της διεργασίας.<br/>

Αλλαγές έπρεπε να γίνουν και στην `usertrap(...)`.
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
Προκειμένου να αντιμετωπιστεί το ενδεχόμενο η εικονική διεύθυνση να μην έχει μεταφραστεί σε φυσική, η συνάρτηση πρέπει να τροποποιηθεί κατάλληλα ώστε να δεσμεύει τον απαιτούμενο χώρο σελίδας και να κάνει το απαραίτητο mapping, αλλά αν το μέγεθος είναι εκτός ορίων τότε η διεργασία θα «σκοτώνεται» απευθείας. Η διαδικασία δέσμευσης και mapping είναι με αυτήν στην συνάρτηση `uvmalloc()` την οποία παροτρύνει η εκφώνηση να χρησιμοποιήσουμε

Αλλαγές έπρεπε να γίνουν και στην `walkaddr(...)`.
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
Για να μπορέσουμε να αντιμετωπίσουμε περίπτωση όπου μία διεργασία περνάει μία έγκυρη διεύθυνση
από την sbrk() σε μία κλήση συστήματος όπως η read() ή write(), αλλά
η μνήμη για τη διεύθυνση αυτή δεν έχει ακόμη ανατεθεί πρέπει να τροποποιήσουμε την συνάρτηση `walkaddr(...)`. Αναλυτικότερα, μετά από μια κλήση συστήματος το kernel δεν θα πάει ποτέ στην `usertrap(...)` οπότε το σφάλμα σελίδας πρέπει να αντιμετωπιστεί στην συνάρτηση `walkaddr(...)` την οποία καλεί τόσο η `copyin(...)` όσο και η `copyout(...)` για να μεταφέρει δεδομένα από user σε kernel και το ανάποδο. Η συνάρτηση προηγουμένως ήλεγχε αν το pte υπάρχει ή είναι valid ή είναι προσβάσιμο από user ώστε επιστραφεί 0 αφού δεν επιτρεπόντουσαν, πλέον στο lazy allocation όλα επιτρέπονται οπότε αρκεί να ισχύει εκ των τριων συνθηκών. Επομένως, αν η εικονική διεύθυνση είναι εντός ορίων τότε εκτελείται η διαδικασία δέσμευσης και mapping διαφορετικά επιστρέφεται 0 το οποίο υποδηλώνει ότι δεν επιτεύχθηκε mapped. Τέλος, αν η δέσμευση και το map εκτελέστηκαν επιτυχώς επιστρέφεται η φυσική διεύθυνση.

  
Επιπλέον αλλαγές έγιναν, τόσο στην συνάρτηση `uvmunmap(...)` όσο και στην `uvmcopy(...)`. 
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
Όσον αφορά τις αλλαγές `uvmunmap(...)` αλλάζουμε την συνάρτηση και την «καλούμε» να αγνοήσει τις σελίδες που είτε δεν έχουν δημιουργηθεί είτε δεν έχουν γίνει mapped ακόμα. Επιπλέον, η `uvmcopy(...)` χρησιμοποιείται , κυρίως, κατα την διαδικασία της `fork()` οπότε πρέπει να τροποποιηθεί καταλλήλως ώστε να αγνοεί πιθανές σελίδες που είτε δεν έχουν δημιουργηθεί είτε δεν έχουν γίνει mapped ακόμα κατά την αντιγραφή πατέρα-παιδιού. Τέλος, παρατηρούμε ότι η `uvmcopy(...)` καλεί την συνάρτηση `uvmunmap(...)` ώστε να διακόψει την αντιστοιχεία του map σε περίπτωση αποτυχίας δέσμευσης μνήμης όμως, έχουμε ήδ τροποποιήσει την συνάρτηση ώστε να αγνοεί τις αντίστοιχες σελίδες.
