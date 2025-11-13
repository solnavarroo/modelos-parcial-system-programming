## **`Funciones auxiliares`**

**Conseguir CR3 desde el selector**
```C
uint32_t mmu_get_cr3_from_selector(int16_t selector){
    int16_t index = selector >> 3;

    gdt_entry_t* taskDescriptor = &gdt[index];

    tss_t* tss = (tts_t*)((taskDescriptor->base_15_0) | 
                        (taskDescriptor->base_23_16 << 16) | 
                        (taskDescriptor->base_31_24 <<24));

    return tss->cr3;
}
```
**Conseguir CR3 desde el indice**
```C
uint32_t cr3 = tss_tasks[i].cr3;
```
**Conseguir PTE desde el selector y virt**
```C
pt_entry_t* mmu_get_pte_for_task(uint16_t selector, vaddr_t virt){
    uint32_t* cr3 = mmu_get_cr3_from_selector(selector);

    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);

    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;

    return (pt_entry_t*) &(pt[pt_index]) ;
}
```
**Conseguir phy desde CR3 y virt, ASUME QUE EXISTE: virt_to_phy, DEVUELVE 0 SI NO EXISTE: obtenerDireccionFisica**
```C
uint32_t virt_to_phy(uint32_t cr3, vaddr_t virt){
    
    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);

    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;

    return (paddr_t) (pt[pt_index].page << 12) ;
}

uint32_t obtenerDireccionFisica(uint32_t cr3,uint32_t* virt){
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    int pdi = VIRT_PAGE_DIR(virt);
    int pti = VIRT_PAGE_TABLE(virt);

    if (!(pd[pdi].attrs & MMU_P)) // Si no está mapeada devuelvo 0
        return 0;

    pt_entry_t* pt = (pt_entry_t*)MMU_ENTRY_PADDR(pd[pdi].pt);
    if (!(pt[pti].attrs & MMU_P)) // Si no está mapeada devuelvo 0
        return 0;

    paddr_t direccion_fisica = MMU_ENTRY_PADDR(pt[pti].page);
    //Hasta acá es CASI IGUAL a mmu_unmap_page (solo cambié nombres de variables), ahora en vez de poner el present en 0, solo devuelvo la dirección física
    return direccion_fisica // OJO esto devuelve la BASE de la página a la que apuntaba la dirección física (sin el offset)
}
```
**EstaMapeada**
```C
bool esta_mapeada(cr3, virt) {
    pd = CR3_TO_PAGE_DIR(cr3);
    if (!(pd[pd_idx].present)) return false;
    
    pt = pd[pd_idx].pt << 12;
    if (!(pt[pt_idx].present)) return false;
    
    return true;
}

bool estaMapeada(vaddr_t virt, task_id tarea){
    uint32_t cr3 = mmu_get_cr3_from_selector(sched_tasks[tarea].selector);
    
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);
    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);
    
    if (!(pd[pd_index].attrs & MMU_P)) 
        return false;
    
    pt_entry_t* pt = (pt_entry_t*)(pd[pd_index].pt << 12);
    
    return (pt[pt_index].attrs & MMU_P);
}
```


**Cambiar tarea**
```asm
cambiarTarea:
    pushad

    call sched_next_task

    mov WORD [sched_task_selector], ax
    jmp far [sched_task_offset]

    popad
    iret

```
**Se quiere chequear un registro no volatil despues de una interrupcion y se mira en la pila**

```C
tss_t* obtener_TSS(uint16_t segsel) {
    uint16_t idx = segsel >> 3;
    return gdt[idx].base;
}

uint8_t es_prioritaria(uint8_t idx) {
    tss_t* tss_task = obtener_TSS(sched_tasks[i].selector);
    uint32_t* pila = tss_task->esp;
    uint32_t edx = pila[5]; //edx esta en la pos 5 d ela pila
    return edx == 0x00FAFAFA;
}
```

**`Funciones y estructuras del TP muy usadas`**

sched.c:

```C
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED
} task_state_t;

// info de la tarea
typedef struct {
  int16_t selector;
  task_state_t state;
} sched_entry_t;

static sched_entry_t sched_tasks[MAX_TASKS] = {0};
```

mmu.c:

```C
typedef struct pd_entry_t {
  uint32_t attrs : 12;
  uint32_t pt : 20;
} __attribute__((packed)) pd_entry_t;

typedef struct pt_entry_t {
  uint32_t attrs : 12;
  uint32_t page : 20;
} __attribute__((packed)) pt_entry_t;

paddr_t mmu_next_free_kernel_page(void)
paddr_t mmu_next_free_user_page(void)

void mmu_map_page(uint32_t cr3, vaddr_t virt, paddr_t phy, uint32_t attrs)
paddr_t mmu_unmap_page(uint32_t cr3, vaddr_t virt)

void copy_page(paddr_t dst_addr, paddr_t src_addr)

static inline void zero_page(paddr_t addr)

bool page_fault_handler(vaddr_t virt) {
  // Chequeemos si el acceso fue dentro del area on-demand
  if(virt >= 0x07000000 && virt <= 0x07000FFF){
  // En caso de que si, mapear la pagina
    print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
    uint32_t cr3 = rcr3();
    mmu_map_page(cr3, 0x07000000, 0x3000000, MMU_W | MMU_U | MMU_P);
    return true;
  }
  return false;
}
```

defines.h:
```h
#define VIRT_PAGE_OFFSET(X) (X & 0x00000FFF)
#define VIRT_PAGE_TABLE(X)  (X & 0x003FF000) >> 12
#define VIRT_PAGE_DIR(X)    (X & 0xFFC00000) >> 22
#define CR3_TO_PAGE_DIR(X)  (X & 0xFFFFF000)
#define MMU_ENTRY_PADDR(X)  (X << 12)
```
