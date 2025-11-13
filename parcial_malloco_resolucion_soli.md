## **2do Parcial de Arquitectura y Organizacion del Computador - Solana Navarro**

**Estructuras nuevas y Modificaciones:**

Necesitamos una estructura que mantenga registro de qué memoria ha reservado cada tarea. Esta estructura DEBE vivir en memoria del kernel, no en memoria de usuario, por varias razones fundamentales. Primero, debe ser accesible desde cualquier contexto del kernel (syscalls, page fault handler, garbage collector) independientemente de qué tarea esté ejecutando. Segundo, debe persistir más allá de la vida de cualquier tarea individual (por ejemplo, cuando una tarea muere, el garbage collector necesita acceder a sus reservas para limpiar). Tercero, si estuviera en memoria de usuario, cada tarea tendría su propia copia y no podríamos coordinar la asignación de memoria globalmente.
```c
static reservas_por_tarea_t reservas[MAX_TAREAS];
```
El array es estático, lo que significa que se aloja en el segmento de datos del kernel y está siempre en memoria. Cada elemento del array corresponde a UNA tarea y contiene TODAS sus reservas. Por ejemplo, reservas[5] contiene todas las reservas de la tarea con task_id 5.

Luego, agrego un nuevo estado de tarea, TASK_BLOCKED, para las tareas que no tiene que volver a correr.

```C
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  TASK_BLOCKED
} task_state_t;
```

¿Por qué TASK_BLOCKED?
Cuando una tarea accede a memoria inválida (fuera de sus reservas), el page fault handler la marca como BLOCKED y la desaloja.
El scheduler NUNCA elegirá una tarea BLOCKED. El scheduler verifica este estado antes de elegir una tarea. Solo las tareas RUNNABLE son candidatas para ejecución. Esto asegura que tareas problemáticas no vuelvan a correr y causen más problemas.
Es diferente de PAUSED porque PAUSED es temporal/reversible, BLOCKED es permanente (la tarea "murió").

## Ejercicio 1:

**a- Cambios necesarios a realizar sobre el kernel**

Antes de que la syscall funcione, debemos registrarla en la IDT (Interrupt Descriptor Table). La IDT es una tabla de 256 entradas que mapea cada número de interrupción a su handler. Para syscalls usamos IDT_ENTRY3, donde el "3" indica que el descriptor tiene DPL=3 (Descriptor Privilege Level 3), lo que significa que código de nivel 3 puede invocarlo. Elijo estos numero porque no se solapan con ninguno del TP.

```C
void idt_init() {
  // ...
  // Nivel 3 porque las tareas de USUARIO tienen que poder usarlas
  IDT_ENTRY3(90);
  IDT_ENTRY3(91);
  //...
}
```

**Syscall de malloco:**

Esta syscall es la encargada de reservar memoria. Consiste en:

La instrucción PUSHAD es crucial: guarda los 8 registros de propósito general en un orden específico (EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI), ocupando 32 bytes totales. Esto nos permite restaurarlos después con POPAD. Sin embargo, hay un truco: necesitamos MODIFICAR el EAX guardado para poner el valor de retorno ahí. Por eso calculamos el offset correcto en la pila (+28 bytes desde el ESP actual después de los pushes) y escribimos directamente en esa posición de memoria.

Asumo que recibe en el registro EDI el tamanio a pedir.

```asm
extern malloco
global isr_90
isr_90:
    pushad

    push edi ; Le entra el tamanio por pila porque estamos en 32 bits
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

Un Page Fault es una excepción (#14) que el CPU lanza cuando intenta traducir una dirección virtual y encuentra que: la página no está presente (bit Present=0), o los permisos son insuficientes (por ejemplo, intento de escritura en página read-only). El procesador automáticamente guarda la dirección problemática en el registro CR2 y salta al handler.
En lazy allocation, los Page Faults son ESPERADOS y NORMALES. No son errores, son el mecanismo por el cual se materializa la memoria reservada. El handler debe distinguir entre "page fault esperado" (dirección en rango reservado → mapear y continuar) y "page fault inesperado" (dirección inválida → desalojar tarea).

Flujo de Manejo
El handler recibe la dirección de CR2. Primero verifica si está en el rango reservable (entre 0xA10C0000 y 0xA10C0000 + 4MB). Si no, es acceso inválido. Si sí, busca específicamente QUÉ reserva contiene esa dirección (porque una tarea puede tener múltiples reservas no contiguas en el espacio lógico, aunque sí lo están virtualmente en nuestro diseño).
Una vez identificada la reserva, pide una página física del pool de usuario (mmu_next_free_user_page). Es crucial ALINEAR la dirección virtual a página antes de mapear, porque el hardware solo trabaja con páginas completas de 4KB. Si virt=0xA10C0678, debemos mapear desde 0xA10C0000. El offset (0x678) se maneja automáticamente por el hardware de paginación.
Después de mapear, inicializa la página a cero (zero_page). Esto es crítico por seguridad: la página física pudo haber contenido datos de otra tarea. Si no la limpiamos, esta tarea podría leer información confidencial. Además, el enunciado especifica comportamiento tipo calloc (memoria inicializada).
Finalmente retorna TRUE, lo que indica al procesador que el problema está solucionado y debe reintentar la instrucción que causó el fault. Ahora la página está mapeada, así que la instrucción funcionará.
Manejo de Accesos Inválidos
Si la dirección no corresponde a ninguna reserva, el handler debe desalojar la tarea permanentemente. Esto involucra: cambiar su estado a TASK_BLOCKED (para que el scheduler nunca más la elija), marcar TODAS sus reservas como pendientes de liberación (estado=2), y cambiar inmediatamente a otra tarea sin retornar a la problemática.

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

Esta es una rutina auxiliar que fuerza un cambio de tarea inmediato, fuera del flujo normal del scheduler. Se usa cuando necesitamos desalojar la tarea actual sin esperar al próximo tick del timer. Hace un JMP FAR, que es la única forma de cambiar de tarea en x86: el procesador guarda automáticamente el estado completo en la TSS actual y carga el estado completo de la nueva TSS.

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

Una TSS de tarea kernel es similar a una de usuario pero con diferencias clave en los segmentos y el page directory. Los selectores de código/datos deben ser de nivel 0 (GDT_CODE_0_SEL, GDT_DATA_0_SEL) en lugar de nivel 3. El CR3 debe apuntar a un page directory con mapeos identity del kernel, no mapeos de usuario. Y ESP0 apunta a una pila de kernel reservada específicamente para esta tarea.
El campo EIP es la dirección donde comienza el código de la tarea. Para el garbage collector, apunta a la función garbage_collector, que contiene un loop infinito. La tarea nunca "termina", simplemente es desalojada periódicamente por el scheduler.

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

La tarea debe tener una entrada en la GDT (para su TSS) y una entrada en el array del scheduler. El proceso es: buscar un slot libre en la GDT (entradas a partir de GDT_TSS_START), crear la TSS con tss_create_kernel_task, crear el descriptor de GDT apuntando a esa TSS, y agregar la tarea al scheduler.

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

La función mmu_init_task_dir_0 crea un page directory vacío y luego mapea todo el rango de memoria del kernel de forma identity. Los permisos son MMU_P | MMU_W (presente y writable) pero SIN MMU_U (no user), lo que significa que solo nivel 0 puede acceder. Esto protege el kernel de accesos accidentales o maliciosos desde nivel 3.

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

Para ejecutar el garbage collector cada 100 ticks, necesitamos un contador global que se incremente en cada interrupción de timer. El scheduler consulta este contador y, cuando es múltiplo de 100, elige al garbage collector en lugar de seguir el round-robin normal.
Este patrón es común para tareas periódicas del sistema: en lugar de tener un thread separado (que requeriría sincronización compleja), simplemente lo tratamos como una tarea más pero con lógica especial en el scheduler.

```C
static uint8_t contador_de_ticks = 0;

void add_tick(){
    contador_de_ticks++;
}
```
El ISR del timer debe llamar a esta función antes de invocar al scheduler, así el contador está actualizado cuando el scheduler toma decisiones.

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

El scheduler ahora tiene dos caminos: si el contador es múltiplo de 100, retorna el selector del garbage collector forzosamente. Si no, procede con el round-robin normal. Esto garantiza que el garbage collector corra periódicamente sin importar qué otras tareas estén activas.
Es importante que el garbage collector NO incremente current_task cuando es elegido de esta forma especial, porque no es parte del round-robin normal. Es una "interrupción" del flujo normal.

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

El garbage collector es un loop infinito que recorre todas las tareas del sistema buscando reservas con estado=2 (pendientes de liberación). Para cada una, desmapea todas las páginas del rango y finalmente marca el estado como 3 (liberada, slot reutilizable).
El proceso de desmapeo es crítico: debe recorrer el rango COMPLETO de la reserva, página por página (incrementos de 4KB), porque lazy allocation pudo haber mapeado solo algunas páginas. Para cada página virtual en el rango, llama a mmu_unmap_page, que: obtiene la página física mapeada, pone Present=0 en la PTE, y retorna la dirección física. En un sistema completo, deberíamos liberar esa página física de vuelta al pool con algo como mmu_free_page(phy), pero el enunciado dice que no nos preocupemos por reciclar.
Es importante usar el CR3 de la tarea víctima (no el nuestro) al desmapear, porque estamos manipulando SU espacio de direcciones, no el del garbage collector. Obtenemos el CR3 desde su TSS con mmu_get_cr3_from_selector.

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

Esta función es fundamental para trabajar con espacios de direcciones de otras tareas. Dado un selector (que identifica una TSS en la GDT), navega: del selector al índice en la GDT, del índice a la entrada en la GDT, de la entrada a la dirección de la TSS (reconstruyendo la base desde los campos fragmentados del descriptor), y de la TSS al campo CR3.

selector → GDT → TSS → CR3.

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

La función malloco es puro código C que corre en nivel 0. Su responsabilidad es RESERVAR un rango de direcciones virtuales, pero NO asignar memoria física. Debe validar varias cosas: que el tamaño no exceda 4MB, que la tarea no haya reservado ya demasiada memoria (suma de todas sus reservas ≤ 4MB), y que haya espacio en el array de reservas.
El cálculo de la dirección virtual es importante: las reservas se concatenan secuencialmente en el espacio virtual. La primera empieza en 0xA10C0000 (dirección arbitraria del enunciado). La segunda empieza donde terminó la primera. Y así sucesivamente. Esto simplifica el manejo porque cada reserva es un rango continuo sin huecos.
Cuando encuentra un slot libre, registra la reserva con estado=1 (activa) y retorna la dirección virtual. CRUCIALMENTE, no llama a mmu_map_page. Las páginas NO están mapeadas todavía. Cuando el usuario intente acceder, habrá un page fault, y ahí el handler mapeará.

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
La función chau implementa este patrón: simplemente cambia el estado de la reserva de 1 (activa) a 2 (pendiente de liberación). El garbage collector, corriendo periódicamente en nivel 0, recorrerá las reservas con estado=2 y hará la liberación real (desmapear páginas, liberar memoria física).

Validaciones de Seguridad
Es crítico validar que la dirección pasada sea válida. El enunciado dice que pasar direcciones inválidas produce "comportamiento indefinido", pero en un sistema real querríamos ser más estrictos. Verificamos: que la dirección esté en una reserva (con esMemoriaReservada), que coincida EXACTAMENTE con el inicio de un bloque (no puede liberar desde el medio), y que el estado sea activo (no puede liberar algo ya liberado).

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
