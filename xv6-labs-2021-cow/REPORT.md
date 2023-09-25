### Implement copy-on-write

O enunciado solicita que seja implementado o mecanismo de copy-on-write (CoW) para o sistema operacional xv6. Para isso, foi criada uma nova flag, dentro dos bits reservados para software do RISC-V (segundo enunciado) para indicar que a página está sendo compartilhada. Essa flag é chamada de `PTE_COW` e é definida no arquivo `kernel/riscv.h`:

```diff
#define PTE_U (1L << 4) // 1 -> user can access
+#define PTE_COW (1L << 8) // 1 -> copy-on-write
```

Também, o struct `kmem` foi modificado para guardar a contagem de referências de cada página, para que seja possível saber quando uma página pode ser liberada:

- `kernel/kalloc.c`

```diff
struct {
  struct spinlock lock;
  struct run *freelist;
+ int reference_count[PHYSTOP/PGSIZE];
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");
+ memset(kmem.reference_count, 0, sizeof(kmem.reference_count) / sizeof(int));
  freerange(end, (void*)PHYSTOP);
}
```

As funções `kfree` e `kalloc` foram modificadas para que a contagem de referências seja atualizada:

- `kernel/kalloc.c`

```diff
// kfree
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

- // Fill with junk to catch dangling refs.
- memset(pa, 1, PGSIZE);
-
- r = (struct run*)pa;
-
  acquire(&kmem.lock);
- r->next = kmem.freelist;
- kmem.freelist = r;
+ // Decrement reference count.
+ kmem.reference_count[PA2REF(pa)]--;
+ if((kmem.reference_count[PA2REF(pa)]) <= 0) {
+   // Fill with junk to catch dangling refs.
+   memset(pa, 1, PGSIZE);
+   r = (struct run*)pa;
+   r->next = kmem.freelist;
+   kmem.freelist = r;
+ }
+
  release(&kmem.lock);

// ...

// kalloc
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
+   kmem.reference_count[PA2REF(r)] = 1;
  }
  release(&kmem.lock);
```

Em seguida, a `usertrap` foi modificada para lidar com page faults. Para isso, foi implementada também uma função `handle_page_fault`. Ambos os códigos comentados seguem:

- `kernel/trap.c`

```diff
+  } else if(r_scause() == 13 || r_scause() == 15){
+    // page fault
+    uint64 va = r_stval();
+
+    // Check if the accessed address is valid and within acceptable ranges.
+    if (va >= MAXVA || va >= p->sz || (va < p->trapframe->sp && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE))
+    {
+      // Mark the process as killed due to an invalid memory access.
+      p->killed = 1;
+    }
+    // Attempt to handle the page fault
+    else if (handle_page_fault(p->pagetable, va) != 0)
+    {
+      // Mark the process as killed if page fault handling fails.
+      p->killed = 1;
+    }
  } ...
```

- `kernel/kalloc.c`

```c
// Handle a page fault at virtual address 'va' in the given 'pagetable'.
int handle_page_fault(pagetable_t pagetable, uint64 va)
{
  if (va >= MAXVA)
    return -1; // Invalid virtual address.

  pte_t *pte = walk(pagetable, va, 0);
  if (pte == 0)
    return -1; // Page table entry not found.

  if ((*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
    return -1; // Invalid or not user-accessible page.

  uint64 mem, pa;
  pa = (uint64)PTE2PA(*pte);

  acquire(&kmem.lock);
  if (kmem.reference_count[PA2REF(pa)] == 1)
  {
    // If reference count is 1, unset COW and set write permission.
    *pte &= ~PTE_COW;
    *pte |= PTE_W;
    release(&kmem.lock);
    return 0; // Successfully handled page fault.
  }
  release(&kmem.lock);

  if ((mem = (uint64)kalloc()) == 0)
    return -1; // Failed to allocate a new page.

  memmove((void *)mem, (void *)pa, PGSIZE); // Copy contents to the new page.
  kfree((void *)pa);                        // Decrease reference count and free 'pa' if necessary.

  // Modify mappings and unset COW.
  *pte = PA2PTE(mem) | PTE_FLAGS(*pte);
  *pte &= ~PTE_COW; // Unset PTE_COW.
  *pte |= PTE_W;  // Set PTE_W (write permission).

  return 0; // Successfully handled page fault.
}
```

Por fim, as funções `uvmcopy` e `copyout` foram modificadas para habilitar o CoW. Também foi implementada uma função `increment_reference_count` para incrementar, de forma segura, a contagem de referências de uma página. Os códigos comentados seguem:

- `kernel/vm.c`

```diff
// uvmcopy
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
-   if((mem = kalloc()) == 0)
-     goto err;
-   memmove(mem, (char*)pa, PGSIZE);
-   if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
-     kfree(mem);
+   flags = (flags | PTE_COW) & (~PTE_W); // Set COW and unset write permission.
+   if(mappages(new, i, PGSIZE, pa, flags) != 0){
      goto err;
    }
+   *pte = PA2PTE(pa) | flags;
+   increment_reference_count((void *)pa);
  }
  return 0;

// ...

// copyout
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
+   if(handle_page_fault(pagetable, va0) != 0)
+     return -1;
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
```

- `kernel/kalloc.c`

```c
// Increment the reference count for a physical address 'pa'.
void increment_reference_count(void *pa)
{
  acquire(&kmem.lock);
  kmem.reference_count[PA2REF(pa)] += 1;
  release(&kmem.lock);
}
```

E assim, finaliza-se o lab.