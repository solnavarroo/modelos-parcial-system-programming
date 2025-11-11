## **Parcial Arquitectura y Organizacion del computador - Solana Navarro**

La idea que tengo es agregar a la estructura de las tareas 2 cosas. Primero, el tipo de acceso que tiene cada tarea. Segundo, la direccion virtual que utilizara. La segunda es mas que nada para los que tienen acceso por copia, ya que su direccion virtual va a ser pasada por parametro, mientras que los que tienen accesoDMA utilizan siempre la direccion virtual 0xBABAB000.


**Estructuras nuevas y Modificaciones:**

Primero, agrego un tipo enumerado para identificar el modo de acceso al buffer:

```C
typedef enum{
    accesoDMA
    accesoCopia
    sinAcceso
}acceso_t
```
Después, extiendo la estructura sched_entry_t (que representa a cada tarea en el scheduler) para tener toda la información:

```C
typedef struct {
  int16_t selector;
  task_state_t state;
  acceso_t acceso;
  vaddr_t dir;
} sched_entry_t;
```
Además, agrego un nuevo estado de tarea, TASK_BLOCKED, para las tareas que están esperando el buffer lleno.
Estas tareas no tienen que ser ejecutadas hasta que el lector de cartuchos indique que el buffer está listo.

```C
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  TASK_BLOCKED
} task_state_t;
```

## Ejercicio 1:

**a- Rutina de interrupción del buffer (ISR 40)**

Primero, en la inicialización de la IDT, agrego la entrada correspondiente a la interrupción 40.
Como se trata de una interrupción de hardware, se ejecutará a nivel 0 por eso utilizo IDT_ENTRY0.

```C
void idt_init() {
  // ...
  IDT_ENTRY0(40); 
  // ...
}
```
La rutina de atención va a guardar el contexto (con pushad), avisa al PIC que la interrupción fue atendida, llama a deviceready, restaurar el contexto(con popad) y retornar.

```asm
extern deviceready
global isr_40
isr_40:
    pushad
    call pic_finish
    call deviceready
    popad
    iret
```

**Función deviceready**

Esta función se ejecuta cada vez que el lector de cartuchos genera una interrupción. Entonces su tarea es recorrer todas las tareas del sistema y, dependiendo de su estado y tipo de acceso, realizar las operaciones necesarias:

- Si la tarea está bloqueada (TASK_BLOCKED):
    - Si tiene acceso DMA, se mapea directamente el buffer físico en su espacio de direcciones.
    - Si tiene acceso por copia, se reserva una nueva página física, se copia el contenido del buffer y se mapea a la dirección virtual que la tarea indicó.
- Si la tarea ya está ejecutándose (TASK_RUNNABLE) y tiene acceso por copia, simplemente se actualiza su copia del buffer.

```C
void deviceready(void){
    for(int i = 0; i < MAX_TAREAS ; i ++){
        sched_entry_t* tarea = &sched_tasks[i];
        uint32_t cr3 = mmu_get_cr3_from_selector(tarea->selector);
        if(tarea->state == TASK_BLOCKED){
            if(tarea->acceso == accesoDMA){
                buffer_dma(CR3_TO_PAGE_DIR(cr3));
            }else if(tarea->acceso == accesoCopia){
                paddr_t direcFisica = mmu_next_user_page();
                buffer_copy(CR3_TO_PAGE_DIR(cr3), direcFisica, tarea.dir);
            }
            tarea->state = TASK_RUNNABLE;
        }else if(tarea->state == TASK_RUNNABLE){
            if(tarea->acceso == accesoCopia){
                paddr_t destino = virt_to_phy(cr3, tarea->dir)
                copy_page((paddr_t)0xF151C000, destino);
            }
        }
    }
}
```

Funciones auxiliares utilizadas:

La funcion auxiliar mmu_get_cr3_from_selector obtiene el valor del CR3 de una tarea, a partir de su selector en la GDT.
Hace: Selector → Descriptor de tarea → TSS → CR3.

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

La funcion auxiliar virt_to_phy es la que hace todo el caminito dese la direccion virtual hasta la fisica. Hace: CR3 → Page Directory → Page Table → Physical Page

```C
uint32_t virt_to_phy(uint32_t cr3, vaddr_t virt){
    
    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);

    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;

    return (paddr_t) (pt[pt_index].page << 12) ;
}
```

**b- Syscalls opendevice y closedevice**

Se agregan a la IDT en nivel 3, ya que son llamadas que deben poder ejecutar las tareas de usuario.

```C
void idt_init() {
  // ...
  IDT_ENTRY3(90);
  IDT_ENTRY3(91); 
  // ...
}
```

**Syscall opendevice:**

Esta syscall bloquea la tarea hasta que el buffer esté listo.
Recibe en el registro ECX la dirección virtual a usar (solo si el acceso es por copia).
El tipo de acceso se lee desde la dirección 0xACCE5000.

```asm
extern opendevice
global isr_90
isr_90:
    pushad

    push ecx ; Le entra la direccion virtual por pila porque estamos en 32 bits
    call opendevice
    add esp, 4 ; restauramos la pila

    call sched_next_task ; Conseguimos proxima tarea

    mov word [sched_task_selector], ax ; la guardamos
    jmp far [sched_task_offset] ; saltamos a ella

    popad
    iret
```
```C
void opendevice(uint32_t dir){
    sched_tasks[current_task].dir = dir;
    sched_tasks[current_task].state = TASK_BLOCKED;
    sched_tasks[current_task].acceso = *(uint8_t*)0xACCE5000;
}
```

**Syscall closedevice:**

Cuando la tarea termina de usar el buffer, debe liberar el mapeo correspondiente.

```asm
extern closedevice
global isr_91
isr_91:
    pushad
    call closedevice
    popad
    iret
```
```C
void closedevice(void){
    if(sched_tasks[current_task].acceso == accesoDMA){
        mmu_unmap_page(rcr3(), (vaddr_t)0xBABAB000);
    }
    if(sched_tasks[current_task].acceso == accesoCopia){
        mmu_unmap_page(rcr3(), sched_tasks[current_task].dir);
    }
    sched_tasks[current_task].acceso = sinAcceso;
}
```

## Ejercicio 2:

**a- buffer_dma:**

Mapea directamente el buffer físico (0xF151C000) en el espacio de direcciones de la tarea, en la dirección virtual fija 0xBABAB000

```C
void buffer_dma(pd_entry_t* pd){
    mmu_map_page((uint32_t)pd, (vaddr_t)0xBABAB000, (paddr_t)0xF151C000, MMU_P | MMU_U);
}
```

**b- buffer_copy:**

Copia en phys el contenido del buffer, y la mapea en la dirección virtual que la tarea indicó.
Este mapeo es lectura/escritura para que la tarea pueda modificar su copia sin afectar el buffer real.

```C
void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt){
    mmu_map_page((uint32_t)pd, virt, phys, MMU_P | MMU_U | MMU_W);
    copy_page(phys, (paddr_t)0xF151C000);
}
```

## EXTRA - Ejercicio 3

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
  tss_tasks[task_id] = tss_create_kernel_task(&killer_main_loop); // Solo cambia esto
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
int8_t current_task = 0;

static uint8_t contador_de_ticks[MAX_TASKS] = {0};

void add_tick_to_task(){
    contador_de_ticks[current_task]++;
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

    call add_tick_to_task

    call sched_next-task

    ;...

    iret
```

La idea de killer_main_loop es que se ejecuta infinitas veces para detectar y deshabilitar tareas de nivel 3 que desperdician recursos del dispositivo. Su ciclo principal recorre todas las tareas activas del sistema y aplica los tres criterios de ociosidad explicados: 
1- que hayan permanecido en estado runnable durante más de 100 ticks de reloj.
2- que mantengan un acceso activo al dispositivo (modo DMA o Copia) y no estén bloqueadas.
3- que no hayan leído la memoria del buffer desde la última inspección.

Para hacer este último punto, el killer obtiene la entrada de tabla de página (PTE) correspondiente a la dirección virtual del buffer y revisa el bit “accessed”. Si el bit permanece en 0, significa que la tarea nunca accedió a esa página de memoria y, por lo tanto, se la marca como TASK_KILLED para liberar recursos y evitar que siga ejecutándose inútilmente. Si el bit está en 1, indica que sí utilizó el buffer; en ese caso se limpia el bit y se reinicia su contador de ticks para volver a observarla más adelante.

```C
void killer_main_loop(void){
    while (true){
        for(int i = 0; i < MAX_TASKS ; i ++){
            if(i == current_task) continue;
            sched_entry_t* task = &sched_tasks[i];

            if(task->state == TASK_PAUSED) continue;
            if(task->acceso == sinAcceso) continue;
            if(task->state == TASK_BLOCKED) continue;
            if(task_ticks[i] <= 100) continue;

            vaddr_t vaddr = 0;

            if(task->acceso == accesoDMA){
                vaddr = VADDR_DMA;
            } else{
                vaddr = task->dir;
            }

            pte_t* pte = mmu_get_pte_for_task(task->selector, vaddr);
            if(pte->accessed == 0){
                task->status = TASK_KILLED;
            }else{
                task_ticks[i] = 0;
                pte->accessed = 0;
            }
        }
    }
}
```

Funcion auxiliar mmu_get_pte_for_task que nos devuelve el puntero a la pt_entry_t

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