## **2do Parcial de Arquitectura y Organizacion del Computador - Solana Navarro**

**Estructuras nuevas y Modificaciones:**

Como dice la consigna tenemos un array de reservas_por_memoria alojado estaticamente en la memoria de el kernel. Entonces defino:

```C
static reservas_por_tarea_t reservas[MAX_TAREAS];
```

Luego, agrego un nuevo estado de tarea, TASK_BLOCKED, para las tareas que no tiene que volver a correr.

```C
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  TASK_BLOCKED
} task_state_t;
```

## Ejercicio 1:

**a- Cambios necesarios a realizar sobre el kernel**

Primero, en la inicialización de la IDT, agrego la entrada correspondiente para la interrupción 90 y 91.
Encambio cuando se habla de la interrupcion 90 y 91, me refiero a malloco (pedir memoria) y a chau (liberar memoria). Entonces como se trata de syscalls, se ejecutará a nivel 3 por eso utilizo IDT_ENTRY3.

```C
void idt_init() {
  // ...
  IDT_ENTRY3(90);
  IDT_ENTRY3(91);
  //...
}
```


**Syscall de malloco:**

Esta syscall es la encargada de reservar memoria
Recibe en el registro EDI el tamanio a pedir.

```asm
extern malloco
global isr_90
isr_90:
    pushad

    push ecx ; Le entra el tamanio por pila porque estamos en 32 bits
    call malloco
    add esp, 4 ; restauramos la pila

    mov [esp + offset_EAX], eax ; la guardamos donde correponde

    popad
    iret
```

**Syscall de chau:**

Esta syscall es la encargada de marcar la tarea como que tiene que ser liberada.

```asm
extern chau
global isr_91
isr_91:
    pushad

    push ecx
    call chau
    add esp, 4

    popad
    iret
```

Cuando una tarea trate de acceder a esa memoria que supuestamente habia reservado, va a saltar un page fault. Por eso lo que hay que hacer es modificar el page fault handler asi recien en ese caso hace el mapeo correspondiente.

- Saber qué tarea falló: current_task (o información del CR3 actual).

- Chequear esMemoriaReservada(virt):

    - Si false → acceso inválido → desalojar la tarea: marcarla bloqueada o TASK_BLOCKED y marcar todas sus reservas para liberar. Luego sched_next_task().

    - Si true → localizar la reserva res tal que virt ∈ [res->virt, res->virt + res->tamanio).

- Asignar sólo la página necesaria:

    - Determinar la dirección de base de página: page_base = ALIGN_DOWN(virt, PAGE_SIZE).

    - Obtener una página física libre del área de usuario: paddr = mmu_next_free_user_page().

    - Mapearla en el directorio de la tarea (usar mmu_map_page(r->cr3, page_base, paddr, flags)).

    - Inicializar física a cero: zero_page(paddr).

    - Importante: no cambies res->estado de toda la reserva — ese campo indica el estado de la reserva global (activa, marcada, liberada). Si quieres registrar qué páginas de esa reserva ya están mapeadas, necesitarías una estructura por-reserva que almacene el bitmap de páginas mapeadas. Pero la consigna no lo pide explícitamente: basta con mapear páginas a medida que caen faults.

- Volver y reintentar la instrucción que falló.

```C

bool page_fault_handler(vaddr_t virt) {

    reservas_por_tarea* reservas = dameReservas(current_task);
    if(esMemoriaReservada(virt)){
        for(int i = 0; i < reservas->reservas_size; i ++){
            reserva_t* reserva = &reservas->array_reservas[i];
            if(virt >= reserva->virt && virt < reserva->virt + reserva->tamanio){
                paddr_t direcFisica = mmu_next_free_user_page();
                mmu_map_page(rcr3(), virt, direcFisica, MMU_P | MMU_U | MM_W);
                zero_page(direcFisica);
                reserva->estado = 1;
                print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
                return true
            }
        }
    }
    sched_task[current_task].state = TASK_BLOCKED;
    for(int i = 0; i < reservas->reservas_size; i ++){
        reserva_t* reserva = &reservas->array_reservas[i];
        reserva->estado = 0;
    }
    cambiarTarea();
    return false;
}
```

```asm
cambiarTarea:
    pushad

    call sched_next_task

    mov WORD [sched_task_selector], ax
    jmp far [sched_task_offset]

    popad
    iret

```

## **Ejercicio 2**

En primer lugar construyo la tss. Explicacion del porque de estos campos:

- CR3: Debe apuntar al page directory que usará la tarea. Para una tarea kernel inicializo el cr3 con un page directory con mapeos de kernel (con mmu_init_task_dir_0).

- EIP: Dirección virtual donde empieza killer_main_loop.

- CS: tiene que ser GDT_CODE_0_SEL para que la tarea se ejecute en nivel 0.

- DS, ES, FS, GS, SS, SS0: Tienen que ser GDT_DATA_0_SEL para que la tarea use los descriptores de datos de nivel 0.

- ESP0: tiene que apuntar al tope de una página de kernel reservada para la pila de nivel 0 de esta tarea (a stack0 + PAGE_SIZE).

- ESP, EBP: como es la pila de usuario, para una tarea kernel que no usa. No importa que pongamos.

```C
tss_t tss_create_kernel_task(paddr_t code_start) {
  //COMPLETAR: es correcta esta llamada a mmu_init_task_dir?
  uint32_t cr3 = mmu_init_task_dir_0(code_start);
  //COMPLETAR: asignar valor inicial de la pila de la tarea
  vaddr_t stack = TASK_STACK_BASE;
  //COMPLETAR: dir. virtual de comienzo del codigo
  vaddr_t code_virt = TASK_CODE_VIRTUAL;
  //COMPLETAR: pedir pagina de kernel para la pila de nivel cero
  vaddr_t stack0 = mmu_next_free_kernel_page();
  //COMPLETAR: a donde deberia apuntar la pila de nivel cero?
  vaddr_t esp0 = stack0 + PAGE_SIZE;
  return (tss_t) {
    .cr3 = cr3,
    .esp = stack0 + PAGE_SIZE,
    .ebp = stack0 + PAGE_SIZE,
    .eip = (vaddr_t) code_start,
    .cs = GDT_CODE_0_SEL,
    .ds = GDT_DATA_0_SEL,
    .es = GDT_DATA_0_SEL,
    .fs = GDT_DATA_0_SEL,
    .gs = GDT_DATA_0_SEL,
    .ss = GDT_DATA_0_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = esp0,
    .eflags = EFLAGS_IF,
  };
}
```
Creo la nueva entrada de TSS en tasks.c:

```C
static int8_t create_task(tipo_e tipo) {
  size_t gdt_id;
  for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
    if (gdt[gdt_id].p == 0) {
      break;
    }
  }
  kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

  int8_t task_id = sched_add_task(gdt_id << 3);
  tss_tasks[task_id] = tss_create_kernel_task(&garbage_collector); // Solo cambia esto
  gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
  return task_id;
}
```

Inicializo tarea con mapeos de kernel en mmu.c:

```C
paddr_t mmu_init_task_dir_0(void){
    paddr_t task_page_dir = mmu_next_free_kernel_page();
    zero_page(task_page_dir);

    for(uint32_t i = 0 ; i < identity_mapping_end ; i += PAGE_SIZE){
        mmu_map_page(task_page_dir, i, i, MMU_W);
    }

    return task_page_dir;
}
```

Inicializo ticks e indico como van a avanzar en sched.c:

```C
static uint8_t contador_de_ticks = 0;

void add_tick(){
    contador_de_ticks++;
}
```

La interrupcion en isr.asm:

```asm
extern add_tick_to_task
global _isr32
_isr32:
    pushad

    call pic_finish1
    call next_clock

    call add_tick

    call sched_next_task

    ;...

    iret
```

```C
uint16_t sched_next_task(void) {
    if(contador_de_ticks % 100 == 0){
        return garbage_collector.selector
    }
    // Buscamos la próxima tarea viva (comenzando en la actual)
    int8_t i;
    for (i = (current_task + 1); (i % MAX_TASKS) != current_task; i++) {
    // Si esta tarea está disponible la ejecutamos
    if (sched_tasks[i % MAX_TASKS].state == TASK_RUNNABLE) {
        break;
    }
    }

    // Ajustamos i para que esté entre 0 y MAX_TASKS-1
    i = i % MAX_TASKS;

    // Si la tarea que encontramos es ejecutable entonces vamos a correrla.
    if (sched_tasks[i].state == TASK_RUNNABLE) {
    current_task = i;
    return sched_tasks[i].selector;
    }

    // En el peor de los casos no hay ninguna tarea viva. Usemos la idle como
    // selector.
    return GDT_IDX_TASK_IDLE << 3;
}
```


```C
void garbage_collector(void){
    while (true){
        for(int i = 0; i < MAX_TASKS ; i ++){
            if(i == current_task) continue;
            sched_entry_t tarea = sched_task[i];
            reservas_por_tarea* reservas = dameReservas(tarea.selector >> 3);
            for(int j = 0; j < reservas->reservas_size; j ++){
                reserva_t* reserva = &reservas->array_reservas[j];
                if(reserva->estado == 2){
                    for(vaddr_t page_addr = reserva->virt; page_addr < reserva->virt + reserva->tamanio; page_addr += PAGE_SIZE){
                        mmu_unmap_page(mmu_get_cr3_from_selector(tarea.selector), page_addr);
                    }
                    reserva->estado = 3;
                }
            }
        }
    }
}
```

Funcion auxiliar mmu_get_cr3_from_selector que nos devuelve el cr3

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

## **Ejercicio 3**

a- La estructura que lleva registros de las reservas estaria definido en memoria de kernel.
Porque es usando por page_fault_handler y el garbage_collector, sin depender del cr3 de la tarea.


b- **Función malloco**

Objetivo: buscar el primer slot libre en array_reservas de la tarea current_task, comprobar que el total reservado + size ≤ 4MB, fijar la dirección virtual de la nueva reserva (empezando en 0xA10C0000 y concatenando reservas anteriores), marcar estado = RESERVA_ACTIVA (1) y devolver el puntero virtual.

```C
#define MAX_MEMORIA (4*1024*1024) // 4MB
void* malloco(size_t size){
    if(size > MAX_MEMORIA){
        return NULL;
    }
    reservas_por_tarea* reservas = dameReservas(current_task);
    if(reservas == NULL){
        return NULL;
    }
    reserva_t* arr = reservas->array_reservas;

    uint32_t i = 0;
    size_t tamanioTotal = 0;
    while(i < reservas->reservas_size && arr[i].estado != 0){
        tamanioTotal += arr[i].tamanio;
        i++;
    }
    if (i == reservas->reservas_size || tamanioTotal + size > MAX_MEMORIA){
        return NULL;
    }
    if(i == 0){
        arr[i].virt = 0xA10C0000;
    }else{
        arr[i].virt = arr[i-1].virt + arr[i-1].tamanio
    }
    
    arr[i].tamanio = size;
    arr[i].estado = 1;

    return (void*) arr[i].virt;
}
```
Objetivo de chau: si virt coincide con el inicio de una reserva activa de la tarea, es marcada para liberar (2). Las condiciones de la consigna dicen que no hace falta comprobar punteros malformados (comportamiento indefinido), pero puedes validar con esMemoriaReservada antes.

```C
void chau(virtaddr_t virt){
    if(esMemoriaReservada(virt)){
        reservas_por_tarea* reservas = dameReservas(current_task);
        if (!reservas) return;
        for(int j = 0; j < reservas->reservas_size; j ++){
            reserva_t* reserva = &reservas->array_reservas[j];
            if(virt == reserva->virt && reserva->estado == 1){
                reserva->estado = 2;
                return;
            }
        }
    }
}
```
