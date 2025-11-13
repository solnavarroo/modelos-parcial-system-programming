
```C
static task_id lock_ocupado = -1;

typedef struct {
  int16_t selector;
  task_state_t state;
  bool esta_esperando_lock; // arranca en false
} sched_entry_t;
```

```C
void get_lock(vaddr_t shared_page){
    if (shared_page == TASK_LOCKABLE_PAGE_VIRT){
        lock_ocupado = current_task;
        sched_tasks[current_task].esta_esperando_lock = false;
        mmu_map_page(tss_tasks[current_task].cr3;, TASK_LOCKABLE_PAGE_VIRT, TASK_LOCKABLE_PAGE_PHY, MMU_P | MMU_W | MMU_U);
        for(int i = 0; i < MAX_TAREAS; i ++){
            if(i == current_task) continue;
            uint32_t cr3 = tss_tasks[i].cr3;
            mmu_unmap_page(cr3, TASK_LOCKABLE_PAGE_VIRT);
        }
    }
}

void lock(vaddr_t shared_page){
    if(current_task == lock_ocupado) return;
    if(lock_ocupado == -1){
        get_lock(shared_page);
    }else if (lock_ocupado != current_task){
        sched_tasks[current_task].esta_esperando_lock = true;
        cambiar_tarea();
    }
}
```

```asm
extern lock
global isr_90
isr_90:
    pushad

    //asumo que la syscall se invoca con la direc virtual en edi
    push edi ; direccion virtual de memoria compartida
    call lock
    add esp, 4

    popad 
    iret
```

```C
void release(vaddr_t shared_page){
    if (shared_page == TASK_LOCKABLE_PAGE_VIRT){
        if(current_task == lock_ocupado){
            lock_ocupado = -1;
            mmu_unmap_page(rcr3(), TASK_LOCKABLE_PAGE_VIRT);
        }
    }
}
```

```asm
extern release
global isr_91
isr_91:
    pushad

    //asumo que la syscall se invoca con la direc virtual en edi
    push edi ; direccion virtual de memoria compartida
    call release
    add esp, 4

    popad 
    iret
```

```c
uint16_t sched_next_task(void) {
  // Buscamos la próxima tarea viva (comenzando en la actual)
  int8_t i;
  for (i = (current_task + 1); (i % MAX_TASKS) != current_task; i++) {
    // Si esta tarea está disponible la ejecutamos
    if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE) {        
        if(sched_tasks[i % MAX_TASKS].esta_esperando_lock){
            if(lock_ocupado == -1){  
                current_task = i % MAX_TASKS;
                get_lock(TASK_LOCKABLE_PAGE_VIRT);
                return sched_tasks[i % MAX_TASKS].selector;
            }
        }else{
            break;
        }
    }
  }

  // Ajustamos i para que esté entre 0 y MAX_TASKS-1
  i = i % MAX_TASKS;

  // Si la tarea que encontramos es ejecutable entonces vamos a correrla.
  if (sched_tasks[i].state == TASK_RUNNABLE) {
    current_task = i;
    return sched_tasks[i].selector;
  }

  // En el peor de los casos no hay ninguna tarea viva. Usemos la idle como selector.
  return GDT_IDX_TASK_IDLE << 3;
}
```

```c
bool page_fault_handler(vaddr_t virt, uint32_t error_code) {
  
    bool fue_escritura = (error_code & 0x2) != 0;

    if (virt == TASK_LOCKABLE_PAGE_VIRT && !(fue_escritura) && tiene_lock == -1){
        mmu_map_page(rcr3(), virt, TASK_LOCKABLE_PAGE_PHY, MMU_U);
        return 1;
    }else if (virt == TASK_LOCKABLE_PAGE_VIRT && fue_escritura){
        return 0;
    }else{
        if (virt >= ON_DEMAND_MEM_START_VIRTUAL && virt <= ON_DEMAND_MEM_END_VIRTUAL) {
            print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
            mmu_map_page(rcr3(), virt, ON_DEMAND_MEM_START_PHYSICAL, MMU_W | MMU_U); // usamos siempre ON_DEMAND_MEM_START_PHYSICAL porque es compartida la memoria y on demand
            return 1; //con esto se arreglo, el tema es que no nos quedaba exactamente 1 en eax por alguna razon, estaba sucio en la parte alta.
        }
        return 0;
    }
}
```
