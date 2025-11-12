```C
void idt_init(){
    // ...
    IDT_ENTRY3(90);
    //...
}
```
```asm
; PushAD Order
%define offset_EAX 28
%define offset_ECX 24
%define offset_EDX 20
%define offset_EBX 16
%define offset_ESP 12
%define offset_EBP 8
%define offset_ESI 4
%define offset_EDI 0

extern espia
global isr_90
isr_90:
    pushad

    ; Selector: AX
    ; Dirección virtual a leer de la tarea espiada: EDI
    ; Dirección virtual a escribir de la tarea espía: ESI
    push esi
    push edi
    push eax
    call espia
    add esp, 16

    mov [esp + offset_EAX], eax

    popad
    iret
```

No tenés acceso a otras direcciones porque cada tarea tiene su propio mapa virtual (paginación), y sólo las direcciones mapeadas en tu CR3 existen para vos.
Las direcciones de otra tarea apuntan a páginas físicas que están en memoria, pero vos no tenés ninguna entrada en tus tablas que las traduzca. La solucion es conseguir la direccion fisicas de esas tareas y mapearlas a la direccion virtual de mi tarea actual.

```C
int8_t espia(int16_t selectorEspiada, vaddr_t virtDeEspiada, vaddr_t virtDeEspia){
    uint32_t cr3Espiada = mmu_get_cr3_from_selector(selectorEspiada);
    paddr_t fisicaEspiada = obtenerDireccionFisica(cr3Espiada, virtDeEspiada);
    if(fisicaEspiada == 0) return 1;

    mmu_map_page(rcr3(), SRC_VIRT_PAGE, fisicaEspiada, MMU_P | MMU_W | MMU_U);
    // Nota: Acá usé SRC_VIRT_PAGE definida en copy_page()
    // Podría usar otra dirección virtual pero es importante usar una reservada para no pisar un mapeo válido 

    uint32_t dato_a_copiar = *((SRC_VIRT_PAGE & 0xFFFFFF000)|VIRT_PAGE_OFFSET(virtDeEspiada)); // Forma la virt con las cosas de src pero el offset que busca la fisica con la virt de la espiada
    
    mmu_unmap_page(rcr3(), SRC_VIRT_PAGE);
    virtDeEspia[0] = dato_a_copiar;

}
```

```C
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

## **Ejercicio 2**

- Cuándo ocurre el cambio de tareas en nuestro sistema?
Durante la interrupción de reloj, _isr32

```asm
_isr32:
    pushad
    call pic_finish1
    call sched_next_task
    ; Si la próxima tarea es 0 o la misma que corre, no saltamos
    cmp ax, 0 ; el scheduler dijo "no cambiamos de tarea"
    je .fin
    str bx ; chequeamos el TR actual
    cmp ax, bx
    je .fin
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]
    .fin:
        popad
        iret
```


- Cómo se elige cuál es la siguiente tarea a ejecutar?
La función sched_next_task decide cuál es la tarea siguiente.

```C
int8_t last_task_priority = 0; // la última tarea con prioridad que ejecutamos (para repartir los turnos entre prioritarias).
int8_t last_task_no_priority = 0; // la última tarea sin prioridad que ejecutamos (para repartir entre las normales)

uint16_t sched_next_task(void) {
    // Buscamos la próxima tarea viva con prioridad
    // Es un for circular que recorre todas las tareas, una por una, empezando desde la siguiente a last_task_priority.
    for ( i = (last_task_priority + 1); (i % MAX_TASKS) != last_task_priority; i++) {
        if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE && es_prioritaria(i)) {
            break; // si encuentra una prioritaria y runnable corta y se queda con ese i
        }
    }

    // A la salida:
        // - i != last_task_priority: más de una tarea prioritaria
        // - i == last_task_priority == current_task, no quiero repetir
        // - i == last_task_priority != current_task, última no fue prioritaria
    
    i = i % MAX_TASKS; //Ajustamos i para que esté entre 0 y MAX_TASKS-1

    if (i != current_task && es_prioritaria(i)) {
        // Hay más de una tarea prioritaria + viva o la última tarea ejecutada fue sin prioridad
        last_task_priority = i; // la ultima priority corrida es la que esta por hacer
        current_task = i;
        return sched_tasks[i].selector; //Para que haga el salto de tareas
    }

    // Si llegué acá es porque
        // - La última tarea ejecutada fue con prioridad (y hay solo una con prioridad)
        // - o no hay tareas con prioridad
        // - o no hay más tareas vivas
    // Busco proxima tarea normal
    for ( i = (last_task_no_priority + 1); (i % MAX_TASKS) != last_task_no_priority; i++) {
        //  Si esta tarea está disponible la ejecutamos
        if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE) {
            break;
        }
    }
    // Si la tarea que encontramos es ejecutable, entonces la corremos.
    if (sched_tasks[i].state == TASK_RUNNABLE) {
        // Si llegamos acá, la tarea que encontramos no es prioritaria
        last_task_no_priority = i;
        current_task = i;
        return sched_tasks[i].selector;
    }
    // En el peor de los casos no hay ninguna tarea viva.
    // Usemos la idle como selector.
    return GDT_IDX_TASK_IDLE << 3;
}

```

**Como identificar tareas prioritarias ( es_prioritaria(i) )??**
Necesitamos revisar el valor de edx que tenía la tarea al momento de ser
interrumpida por el clock. La tarea no está actualmente en ejecución, entonces dónde está su información? En la TSS.

Qué es la TSS? En qué momento se carga? En qué momento se actualiza?

• La TSS se actualiza cuando hacemos jmp far ( jmp sel:offset ), por lo que en la TSS se guardan los valores de los registros al momento del jmp.

edx es un registro no volátil, por lo que si recordamos la rutina de reloj de antes, después de los llamados a funciones de C ( sched_next_task ) es probable que no tenga el mismo valor que nos había llegado.

Es decir, TSS.edx no tiene el valor que buscamos.

Sin embargo, el valor de edx que nos interesa vive aún. Dónde? en la pila.

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
