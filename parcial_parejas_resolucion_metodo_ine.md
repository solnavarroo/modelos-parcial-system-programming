
Defino una variable estatica.

```C
static pareja_t* parejas[MAX_TAREAS] = {0}
```

```C
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED, 
  TASK_BLOCKED
} task_state_t;

typedef struct {
  task_id lider;
  task_id participante;
} pareja_t;

```

```asm
extern crear_pareja
global isr_90
isr_90:
    pushad
    call crear_pareja
    popad
    iret
```

```asm
extern juntarse_con
global isr_91
isr_91:
    pushad
    push edi
    call juntarse_con
    add esp, 4

    mov [esp + offset_eax], eax

    popad
    iret
```

```asm
extern abandonar_pareja
global isr_92
isr_92:
    pushad
    call abandonar_pareja
    popad
    iret
```

```C
void crear_pareja(){
    if(parejas[current_task] != 0) return;
    sched_task[current_task].state = TASK_BLOCKED;
    pareja_t* nuevaPareja = malloc(sizeof(pareja_t));
    nuevaPareja->lider = current_task;
    parejas[current_task] = nuevaPareja;
    cambiar_tarea();
}
```


```C
int juntarse_con(int id_tarea){
    if(aceptando_pareja(current_task) && aceptando_pareja(id_tarea)){
        conformar_pareja(id_tarea);
        return 0;
    }
    return 1;
}
```

```C
void abandonar_pareja(){
    if(parejas[current_task] == 0) return;
    for(vaddr_t virt = 0xC0C00000; virt < 0xC0C00000 + 4MB; virt += PAGE_SIZE){
        if(estaMapeadaPorPareja(virt, current_task)){
            mmu_unmap_page(rcr3(), virt);
        }
    }
    romper_pareja();
}
```

```C
bool page_fault_handler(vaddr_t virt) {
    if(virt >= (vaddr_t)0xC0C00000 && virt < (vaddr_t)0xC0C00000 + 4MB){
        if(pareja_de_actual() != 0){
            paddr_t direcFisica = mmu_next_free_user_page();
            uint32_t cr3 = mmu_get_cr3_from_selector(sched_tasks[pareja_de_actual()].selector);
            if(es_lider(current_task)){
                mmu_map_page(rcr3(), virt, direcFisica, MMU_P | MMU_W | MMU_U);
                mmu_map_page(cr3, virt, direcFisica, MMU_P | MMU_U);
            }else{
                mmu_map_page(rcr3(), virt, direcFisica, MMU_P | MMU_U);
                mmu_map_page(cr3, virt, direcFisica, MMU_P | MMU_W | MMU_U);
            }
            zero_page(virt);
            return true;
        }
    }

    
    // Chequeemos si el acceso fue dentro del area on-demand
    if(virt >= 0x07000000 && virt <= 0x07000FFF){
        // En caso de que si, mapear la pagina
        uint32_t cr3 = rcr3();
        print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
        mmu_map_page(cr3, 0x07000000, 0x3000000, MMU_W | MMU_U | MMU_P);
        return true;
    }
    return false;
}
```

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
**Ejercicio 2**
```C
void conformar_pareja(task_id tarea){
    sched_task[id_tarea].state = TASK_RUNNABLE;
    parejas[id_tarea]->participante = current_task;
    parejas[current_task] = parejas[id_tarea];
}

void romper_pareja(){
    if(es_lider(current_task)){
        if(parejas[current_task]->participante != 0){
            sched_task[current_task].state = TASK_BLOCKED;
            cambiar_tarea();
        }else{
            parejas[current_task] = 0;
        }
    }else{
        if(sched_tasks[pareja_de_actual()].state == TASK_BLOCKED){
            parejas[pareja_de_actual()] = 0;
            sched_tasks[pareja_de_actual()].state = TASK_RUNNABLE;
        }else{
            parejas[pareja_de_actual()]->participante = 0;
        }
        parejas[current_task] = 0;
    }
}

```

**Ejercicio 3**
```C
task_id pareja_de_actual(){
    if(es_lider(current_task)){
        return parejas[current_task]->participante;
    }
    return parejas[current_task]->lider;
}

bool es_lider(task_id tarea){
    return parejas[current_task]->lider == tarea
}

bool aceptando_pareja(task_id tarea){
    if(parejas[tarea] == 0 || parejas[tarea]->participante) return 1;
    return 0;
}
```

## **Parte 2: monitoreo de la memoria de las parejas**
```C
uint32_t uso_de_memoria_de_las_parejas(){
    uint32_t cantidad = 0;
    uint8_t fue_contado[MAX_TAREAS] = {0};

    for(int i = 0; i < MAX_TAREAS; i++){
        task_id lider = parejas[i]->lider;
        task_id participante = parejas[i]->participante;
        if(!fue_contado[lider]){
            for(vaddr_t virt = 0xC0C00000; virt < 0xC0C00000 + 4MB; virt += PAGE_SIZE){
                if(estaMapeada(virt, lider) || estaMapeada(virt, participante)){
                    cantidad +=1;
                }
            }
            fue_contado[lider] = 1;
            fue_contado[participante] = 1;
        }
    }
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


