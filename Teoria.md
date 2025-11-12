## **Teoría Completa - Sistema Operativo x86 en Modo Protegido**

## NIVELES DE PRIVILEGIO (RINGS)
Concepto Fundamental
El procesador x86 implementa 4 niveles de privilegio (rings 0-3), pero en la práctica solo usamos 2:
Ring 0 (Nivel 0) - Kernel/Supervisor:
El nivel más privilegiado. Aquí corre el código del sistema operativo. Tiene acceso completo a todos los recursos del hardware: puede ejecutar instrucciones privilegiadas, acceder a toda la memoria, modificar registros de control (CR0, CR3, CR4), manipular tablas del sistema (GDT, IDT), y controlar dispositivos de E/S. El kernel es el único que puede gestionar la memoria, el scheduler, y las interrupciones. Es la capa de confianza absoluta del sistema.
Ring 3 (Nivel 3) - Usuario/Aplicación:
El nivel menos privilegiado. Aquí corren las aplicaciones de usuario. Tiene restricciones importantes: no puede ejecutar instrucciones privilegiadas, solo puede acceder a memoria explícitamente mapeada con el bit U/S=1, no puede modificar estructuras del sistema, y debe solicitar servicios al kernel mediante syscalls. Esto protege al sistema de errores o código malicioso en aplicaciones.
Por Qué Esta Separación
La separación de privilegios es fundamental para la estabilidad y seguridad del sistema. Si una aplicación de usuario pudiera modificar directamente las tablas de paginación o la GDT, podría crashear todo el sistema o acceder a memoria de otras aplicaciones. El kernel actúa como intermediario confiable que valida todas las operaciones sensibles. Cuando una aplicación necesita hacer algo privilegiado (leer un archivo, reservar memoria, comunicarse con hardware), no lo hace directamente: pide permiso al kernel mediante una syscall, el kernel valida la operación, y si es válida, la ejecuta en nombre de la aplicación.
Cómo Se Determina el Nivel Actual
El nivel de privilegio actual está determinado por el selector de segmento CS (Code Segment). Los últimos 2 bits del selector (bits 0-1) indican el RPL (Requested Privilege Level). Cuando el procesador ejecuta código, verifica el RPL del CS contra el DPL (Descriptor Privilege Level) de los recursos accedidos. Si RPL > DPL, el acceso se deniega con una excepción de protección general (#GP).
Los selectores típicos son:
GDT_CODE_0_SEL = 0x08  (binario: ...01000)  → RPL = 0 (kernel)
GDT_CODE_3_SEL = 0x1B  (binario: ...11011)  → RPL = 3 (usuario)

## CAMBIOS DE CONTEXTO Y PRIVILEGIO
Task Switch (Cambio de Tarea)
El cambio de tarea en x86 se realiza mediante la instrucción JMP FAR a un selector de TSS. Este es un mecanismo completo de hardware que automáticamente: guarda el estado completo del procesador (todos los registros, EFLAGS, EIP, segmentos) en la TSS de la tarea saliente; carga el CR3 de la nueva tarea (cambiando todo el espacio de direcciones virtuales); restaura todos los registros desde la TSS de la tarea entrante; y actualiza el registro TR (Task Register) para apuntar a la nueva TSS.
Este proceso es completamente atómico: el procesador garantiza que no puede interrumpirse a mitad de camino. Esto es crítico porque durante el cambio de contexto el sistema está en un estado inconsistente. La TSS actúa como "snapshot" del estado de una tarea cuando no está ejecutando.
Privilege Level Switch (Cambio de Nivel)
Un cambio de nivel de privilegio ocurre típicamente en dos situaciones: syscalls (3→0) e interrupciones desde usuario (3→0), y al retornar de ellas (0→3).
Cuando se pasa de nivel 3 a nivel 0:

El procesador detecta que el DPL del descriptor objetivo (interrupt gate, trap gate) es 0
Automáticamente busca en la TSS actual los campos ESP0 y SS0
Cambia al stack de kernel (ESP0) para no contaminar el stack de usuario
Pushea en el nuevo stack de kernel: SS usuario, ESP usuario, EFLAGS, CS usuario, EIP usuario
Si hay error code (algunas excepciones): también lo pushea
Carga CS:EIP del descriptor de la IDT
Ahora está en nivel 0 con stack de kernel

Cuando se retorna de nivel 0 a nivel 3:
La instrucción IRET realiza el proceso inverso: popea EIP, CS, EFLAGS del stack; si CS tiene RPL=3, también popea ESP y SS (stack de usuario); y finalmente restaura la ejecución en nivel 3 con el stack original de usuario.
Esta mecánica de doble stack es crucial: si usáramos el mismo stack para kernel y usuario, una aplicación maliciosa podría corromper datos del kernel en su propio stack.
Por Qué JMP FAR para Tareas
La instrucción JMP FAR es especial porque incluye tanto un offset como un selector. Cuando el selector apunta a una TSS (tipo 0x9 en GDT), el procesador interpreta esto como "cambiar de tarea" en lugar de un simple salto. No es que "necesitemos" JMP FAR, es que es el único mecanismo que x86 provee para cambio de contexto por hardware. La alternativa sería implementar cambio de contexto por software (guardando y restaurando manualmente todos los registros), pero sería más lento y propenso a errores.

## SYSCALLS - System Calls
Qué Son
Las syscalls son el puente entre nivel 3 y nivel 0. Son la única forma que tiene una aplicación de usuario de solicitar servicios privilegiados al kernel. Son como "funciones del kernel" que pueden llamarse desde usuario, pero con una interfaz y protecciones especiales.
Ejemplos comunes: abrir/leer/escribir archivos, reservar/liberar memoria, crear procesos, comunicación entre procesos, acceso a dispositivos, obtener hora del sistema, etc. Sin syscalls, una aplicación estaría completamente aislada y no podría hacer nada útil.
Mecanismo de Invocación
Las syscalls se invocan mediante interrupciones de software usando la instrucción INT n. Por ejemplo, INT 0x90 podría ser la syscall "malloco", INT 0x91 podría ser "chau", etc. La elección del número es arbitraria pero debe configurarse en la IDT.
Cuando se ejecuta INT 0x90:

El procesador busca la entrada 0x90 en la IDT
Encuentra un interrupt gate con DPL=3 (accesible desde usuario)
Cambia a nivel 0, usando ESP0 de la TSS
Salta al handler configurado (ISR_90)
El handler en assembly prepara parámetros y llama a la función C del kernel
La función C ejecuta la lógica de la syscall
Se retorna con IRET, volviendo a nivel 3

Pasaje de Parámetros
Los parámetros se pasan mediante registros porque es el mecanismo más eficiente. La convención típica en 32 bits es:

EDI, ESI, EDX, ECX, EBX: parámetros de entrada (en orden)
EAX: valor de retorno
Stack: para parámetros adicionales si son necesarios

En el ISR de assembly, hacemos pushad para preservar todos los registros, luego pusheamos los parámetros específicos a la pila en orden inverso (porque el stack crece hacia abajo), llamamos a la función C, y finalmente limpiamos la pila con add esp, N.
Por ejemplo, para espia(selector, virt_leer, virt_escribir):
asmpushad                 ; Guardar todos los registros
push esi              ; 3er parámetro (virt_escribir)
push edi              ; 2do parámetro (virt_leer)
push eax              ; 1er parámetro (selector)
call espia            ; Llamar función C
add esp, 12           ; Limpiar pila (3 params × 4 bytes)
mov [esp+28], eax     ; Guardar retorno en EAX de pushad
popad                 ; Restaurar registros
iret                  ; Volver a usuario
Por Qué IDT_ENTRY3 vs IDT_ENTRY0
Las syscalls siempre usan IDT_ENTRY3 porque deben ser invocables desde nivel 3 (usuario). El "3" en IDT_ENTRY3 indica el DPL (Descriptor Privilege Level) del interrupt gate. Un gate con DPL=3 puede invocarse desde cualquier nivel (0, 1, 2, 3). Un gate con DPL=0 solo puede invocarse desde nivel 0.
Si pusiéramos una syscall con IDT_ENTRY0, cuando una tarea de usuario hiciera INT 0x90, el procesador lanzaría una excepción #GP (General Protection Fault) porque estaría violando los niveles de privilegio. No es que "no funcione", es que el procesador activamente lo previene por seguridad.
Las interrupciones de hardware (IRQs) usan IDT_ENTRY0 porque son generadas por el hardware externo, no por código de usuario. El procesador las maneja automáticamente sin verificar privilegios de la "tarea" (porque no hay tarea invocante, es un evento externo).

## INTERRUPCIONES Y EXCEPCIONES
Diferencias Fundamentales
Excepciones: Son eventos síncronos generados por el procesador durante la ejecución de instrucciones. Ejemplos: Division by Zero (#DE, int 0), Page Fault (#PF, int 14), General Protection Fault (#GP, int 13), Invalid Opcode (#UD, int 6). Son "problemas" que surgen al ejecutar código. Algunas son recuperables (page fault en lazy allocation), otras son fatales (división por cero no manejada).
Interrupciones de Hardware (IRQs): Son eventos asíncronos generados por dispositivos externos. Ejemplos: Timer (IRQ 0 → int 32), Teclado (IRQ 1 → int 33), Disco (IRQ 14), Red (IRQ 11). Ocurren independientemente de lo que esté ejecutando el procesador. El timer es especialmente importante para implementar preemptive multitasking.
Interrupciones de Software: Son invocadas explícitamente por el código mediante INT n. No son realmente "interrupciones" en el sentido de eventos inesperados, son invocaciones deliberadas. Se usan principalmente para syscalls.
Page Fault Handler - Caso Especial
El Page Fault es la excepción #14 y es particularmente interesante porque puede ser tanto un error como algo esperado. Ocurre cuando:

Se intenta acceder a memoria con bit Present=0
Se intenta escribir en página con R/W=0
Se intenta acceder desde nivel 3 a página con U/S=0

El procesador automáticamente guarda la dirección que causó el fallo en el registro CR2. También pushea un error code en el stack que indica el tipo de fallo (lectura vs escritura, usuario vs supervisor, protección vs no presente).
En sistemas modernos, el page fault es la base de muchas optimizaciones: lazy allocation (reservar sin mapear), copy-on-write (compartir hasta escritura), demand paging (cargar páginas bajo demanda), memory-mapped files, etc. El handler debe distinguir entre "page fault esperado" (manejar y retornar true) y "page fault inválido" (desalojar tarea).
PIC y pic_finish
El PIC (Programmable Interrupt Controller) es el chip que gestiona interrupciones de hardware. Cuando un dispositivo genera un IRQ, el PIC lo traduce a un número de interrupción y notifica al procesador. El procesador ejecuta el handler correspondiente.
Crucialmente, el PIC espera un EOI (End Of Interrupt) del software antes de procesar más interrupciones. Por eso siempre debemos llamar pic_finish() o pic_finish1() al principio de un IRQ handler. Si olvidamos esto, el PIC quedará "colgado" esperando el EOI y no procesará más interrupciones de ese nivel o inferior.
pic_finish1() se usa para el PIC maestro (IRQs 0-7), pic_finish2() para el esclavo (IRQs 8-15). En sistemas con un solo PIC o con PIC configurado en cascada, normalmente basta con pic_finish1().

## PAGINACIÓN Y MEMORIA VIRTUAL
Concepto General
La paginación es el mecanismo por el cual cada tarea tiene su propio espacio de direcciones virtuales independiente. Dos tareas pueden usar la misma dirección virtual (por ejemplo, 0x08000000) pero acceder a memoria física completamente diferente. Esto proporciona:
Aislamiento: Una tarea no puede acceder a memoria de otra (salvo que se mapee explícitamente).
Flexibilidad: Cada tarea puede tener su memoria organizada de forma lógica sin preocuparse por la memoria física disponible.
Protección: El kernel puede controlar qué memoria es accesible y con qué permisos (read-only, read-write).
Compartición controlada: Múltiples tareas pueden compartir memoria mapeando la misma página física en sus espacios virtuales.
Traducción de Direcciones
Cada dirección virtual de 32 bits se divide en:

Bits 31-22 (10 bits): Índice en el Page Directory (1024 entradas)
Bits 21-12 (10 bits): Índice en la Page Table (1024 entradas)
Bits 11-0 (12 bits): Offset dentro de la página (4096 bytes)

El proceso de traducción es:

Leer CR3 para obtener la base del Page Directory
Usar bits 31-22 para indexar el PD y obtener una PDE
Verificar bit Present de la PDE; si es 0, Page Fault
Usar el campo "pt" de la PDE (bits 31-12) como base de la Page Table
Usar bits 21-12 para indexar la PT y obtener una PTE
Verificar bit Present de la PTE; si es 0, Page Fault
Usar el campo "page" de la PTE (bits 31-12) como base de la página física
Agregar el offset (bits 11-0) para obtener la dirección física final

Este proceso lo hace el hardware automáticamente en cada acceso a memoria. Es transparente para el código que se ejecuta. Solo falla (Page Fault) si encuentra Present=0 o viola permisos.
CR3 - El Registro Mágico
El registro CR3 contiene la dirección física del Page Directory de la tarea actual. Cada tarea tiene su propio Page Directory (y por ende su propio CR3). Cuando se hace un cambio de tarea mediante JMP FAR a una TSS, el procesador automáticamente carga el CR3 de la nueva tarea, cambiando instantáneamente todo el espacio de direcciones virtuales.
Por eso dos tareas pueden ejecutar exactamente el mismo código (mismas direcciones virtuales) pero operar sobre datos completamente diferentes. El CR3 es el "switch" que determina qué mapa de memoria se está usando.
Es importante entender que el CR3 almacena una dirección física, no virtual. Tiene que ser así porque se usa para traducir direcciones virtuales; si el propio CR3 fuera virtual, tendríamos un problema de "huevo y gallina".
Identity Mapping para Kernel
El código del kernel típicamente usa identity mapping: la dirección virtual es igual a la dirección física. Por ejemplo, el código en 0x00001000 física se mapea a 0x00001000 virtual. Esto simplifica enormemente el código del kernel porque puede usar direcciones físicas directamente sin preocuparse por traducción.
Sin embargo, las tareas de usuario NO tienen identity mapping. Su código podría estar en 0x08000000 virtual pero en 0x01234000 física. Esto permite que múltiples tareas coexistan sin colisionar en memoria física.
Cuando el kernel necesita acceder a memoria física directamente (por ejemplo, para configurar una nueva Page Table), usa funciones como mmu_map_page() que trabajan con direcciones físicas pero las mapean temporalmente en el espacio virtual del kernel.
Compartir Memoria entre Tareas
Para que dos tareas compartan memoria, simplemente mapeamos la misma página física en ambos espacios virtuales. Por ejemplo:

Tarea A: mapea virtual 0xC0C00000 → física 0x02000000
Tarea B: mapea virtual 0xC0C00000 → física 0x02000000

Ahora ambas tareas acceden a la misma memoria física, aunque cada una usa su propio CR3. Los cambios que una haga son visibles para la otra. Esto es la base de memoria compartida, comunicación entre procesos, etc.
Los permisos pueden ser diferentes: la tarea A podría tener MMU_W (write) mientras que B solo tiene read-only. Esto permite esquemas como "productor-consumidor" donde uno escribe y otros solo leen.
Por Qué Alinear a 0xFFFFF000
Las páginas en x86 son de 4KB (0x1000 bytes). El hardware de paginación trabaja con páginas completas, no con bytes individuales. Los bits 11-0 de las direcciones en PDE/PTE son atributos, no dirección. Los bits 31-12 son el número de página.
Cuando mapeamos, debemos mapear desde el inicio de una página. Si intentamos mapear 0x12345678, el hardware lo interpretaría como "página 0x12345" con atributos corruptos 0x678. Por eso siempre hacemos virt & 0xFFFFF000 para redondear hacia abajo al inicio de la página.
El offset (bits 11-0) se maneja automáticamente: si accedemos a 0x12345678, el hardware traduce la página 0x12345 y luego agrega el offset 0x678 en la física resultante.

## MMU - Memory Management Unit
Funciones del Kernel
Las funciones mmu_* son código del kernel que manipula las estructuras de paginación. Corren en nivel 0 porque necesitan:

Acceder a Page Directories y Page Tables de cualquier tarea
Modificar entradas (mapear/desmapear)
Reservar páginas físicas del pool global
Manipular CR3 de otras tareas (vía TSS)

mmu_map_page
Esta función crea un mapeo: virtual → física con ciertos atributos. Los pasos son:

Recibe un CR3 (de qué tarea), dirección virtual, dirección física, y flags
Calcula índices PD y PT desde la virtual
Obtiene la PDE correspondiente
Si la PT no existe (Present=0 en PDE), crea una nueva PT (reserva página física, la inicializa a cero, la registra en la PDE)
Obtiene la PTE correspondiente en esa PT
Configura la PTE: bits 31-12 = número de página física, bits 11-0 = atributos (Present, User, Write, etc.)
Invalida TLB si es necesario (para que el procesador recargue la traducción)

Es crucial que esta función sea atómica o se llame con interrupciones deshabilitadas cuando se opera sobre estructuras compartidas.
mmu_unmap_page
Hace lo opuesto: elimina un mapeo. Los pasos son:

Navega por PD → PT → PTE (similar a mmu_map_page)
Lee la dirección física de la PTE (para retornarla, si el llamador quiere liberar esa física)
Pone Present=0 en la PTE
Opcionalmente podría liberar la PT si quedó completamente vacía (optimización)
Invalida TLB

Después de unmapear, cualquier acceso a esa dirección virtual causará Page Fault.
mmu_next_free_user_page / mmu_next_free_kernel_page
Estas funciones reservan una página física del pool global. El sistema mantiene un contador de qué páginas están libres (típicamente un bitmap o simplemente un puntero que avanza). Devuelven la dirección física de una página libre.
mmu_next_free_user_page reserva de un área destinada a tareas de usuario (típicamente memoria alta). mmu_next_free_kernel_page reserva de un área destinada al kernel (típicamente memoria baja). Esta separación evita fragmentación y facilita debugging.
Una vez reservada, la página NO se devuelve automáticamente. El sistema debe llevar registro de qué páginas están en uso. En un sistema real, habría una función mmu_free_page(paddr) para devolver páginas al pool.
zero_page
Inicializa una página física a ceros. Recibe una dirección virtual (no física directamente). Esto es porque el procesador no puede acceder a direcciones físicas directamente; necesita que estén mapeadas en el espacio virtual.
Típicamente hace:
cvoid zero_page(vaddr_t virt) {
    memset((void*)virt, 0, 4096);
}
Es crucial para seguridad: cuando asignamos memoria nueva a una tarea, debe estar en cero para que no lea datos de otra tarea que usó esa página antes.
copy_page
Copia contenido de una página física a otra. Similar a zero_page, trabaja con direcciones virtuales. Típicamente mapea temporalmente ambas páginas físicas en direcciones virtuales conocidas del kernel, hace un memcpy, y luego las desmapea.
cvoid copy_page(paddr_t dst, paddr_t src) {
    // Mapear temporalmente
    mmu_map_page(kernel_cr3, TEMP_VIRT_DST, dst, MMU_P | MMU_W);
    mmu_map_page(kernel_cr3, TEMP_VIRT_SRC, src, MMU_P);
    
    // Copiar
    memcpy((void*)TEMP_VIRT_DST, (void*)TEMP_VIRT_SRC, 4096);
    
    // Desmapear
    mmu_unmap_page(kernel_cr3, TEMP_VIRT_DST);
    mmu_unmap_page(kernel_cr3, TEMP_VIRT_SRC);
}

## SCHEDULER
Qué Es y Por Qué Existe
El scheduler es el componente del kernel que decide qué tarea ejecutar en cada momento. Sin scheduler, solo podríamos correr una tarea a la vez hasta que termine. Con scheduler, podemos tener multitasking: múltiples tareas "corriendo simultáneamente" (en realidad, turnándose muy rápido).
El scheduler es invocado típicamente por:

Timer interrupt (IRQ 0): Cada cierto tiempo (por ejemplo, cada 10ms), el timer genera una interrupción. El handler llama al scheduler para ver si es momento de cambiar de tarea.
Syscalls que bloquean: Si una tarea llama a una syscall que no puede completarse inmediatamente (por ejemplo, esperar un lock), la syscall misma llama al scheduler para cambiar a otra tarea.
Excepciones: Si una tarea causa un error fatal, el scheduler la marca como muerta y elige otra.

Estados de Tareas
Una tarea puede estar en varios estados:

TASK_RUNNABLE: Lista para ejecutar. El scheduler puede elegirla en cualquier momento.
TASK_PAUSED: Pausada manualmente (por debug, por ejemplo). El scheduler la salta.
TASK_BLOCKED: Esperando algún evento (lock, I/O, timer, etc.). No debe ejecutarse hasta que el evento ocurra.
TASK_KILLED: Terminada o desalojada permanentemente. Nunca vuelve a ejecutarse.

El scheduler solo elige tareas en estado RUNNABLE. Cuando un evento ocurre (por ejemplo, un lock se libera), el código correspondiente cambia el estado de las tareas esperando de BLOCKED a RUNNABLE.
Round-Robin Básico
Es el algoritmo de scheduling más simple. Mantiene una lista de tareas RUNNABLE y las ejecuta en orden circular. Cada tarea corre por un quantum (período fijo de tiempo, determinado por el timer). Cuando expira el quantum, se pasa a la siguiente tarea.
cuint16_t sched_next_task(void) {
    // Buscar siguiente tarea RUNNABLE
    for (int i = current_task + 1; i != current_task; i = (i+1) % MAX_TASKS) {
        if (sched_tasks[i].state == TASK_RUNNABLE) {
            current_task = i;
            return sched_tasks[i].selector;
        }
    }
    // Si no hay ninguna, devolver idle
    return IDLE_SELECTOR;
}
Es justo (todas las tareas reciben el mismo tiempo) pero no considera prioridades.
Scheduler con Prioridades
Un refinamiento es distinguir tareas prioritarias de normales. El algoritmo típico:

Si hay tareas prioritarias RUNNABLE, elegir la siguiente (round-robin entre prioritarias)
Si no hay prioritarias, elegir la siguiente tarea normal (round-robin entre normales)
Si no hay ninguna, idle

Esto asegura que tareas importantes (por ejemplo, drivers de hardware) siempre corran antes que tareas de fondo (por ejemplo, cálculos).
La dificultad está en detectar qué tareas son prioritarias. Puede ser un campo explícito en la estructura de la tarea, o dinámico (por ejemplo, chequeando un registro específico al momento de la interrupción de reloj).
current_task
Es una variable global del kernel que indica el índice de la tarea actualmente en ejecución. Se usa constantemente:

En syscalls, para saber quién invocó la syscall
En page faults, para saber quién causó el fallo
En el scheduler, para saber cuál es la "tarea actual" que está cediendo el control

Es crítico mantenerla actualizada: cada vez que se hace un cambio de tarea, debe actualizarse current_task antes del JMP FAR (o inmediatamente después al retomar ejecución).

## GDT, TSS, IDT
GDT - Global Descriptor Table
La GDT es una tabla en memoria que contiene descriptores de segmentos. Cada descriptor define un segmento: su base, límite, tipo, y nivel de privilegio. En modo protegido, todos los accesos a memoria pasan por segmentos.
Los selectores (valores cargados en CS, DS, SS, etc.) son simplemente índices en la GDT (más algunos flags). Por ejemplo, selector 0x08 significa "entrada 1 de la GDT" (porque los últimos 3 bits son flags).
Para nuestro sistema típico, la GDT contiene:

Entrada 0: NULL (no usable, por especificación x86)
Entrada 1: Code segment nivel 0 (kernel)
Entrada 2: Data segment nivel 0 (kernel)
Entrada 3: Code segment nivel 3 (usuario)
Entrada 4: Data segment nivel 3 (usuario)
Entradas 5+: TSS para cada tarea

Los segmentos de código/datos típicamente abarcan toda la memoria (4GB) y se dejan los límites en 0xFFFFF con granularidad de 4KB. Esto efectivamente "desactiva" la segmentación, dejando solo la paginación como mecanismo de protección. Es la configuración más común en sistemas modernos.
TSS - Task State Segment
La TSS es una estructura que almacena el estado completo de una tarea. Cada tarea tiene su propia TSS. Contiene: todos los registros de propósito general (EAX, EBX, etc.), registros de segmento (CS, DS, etc.), punteros de pila (ESP, EBP), CR3 (page directory), EIP (instruction pointer), EFLAGS, y punteros a pilas de otros niveles (ESP0, SS0 para nivel 0).
Cuando se hace un JMP FAR a un selector de TSS, el procesador automáticamente guarda el estado actual en la TSS "saliente" y carga el estado de la TSS "entrante". Esto es el cambio de contexto por hardware.
Cada TSS debe tener una entrada en la GDT con tipo "TSS disponible" (0x9). El selector de esa entrada es lo que se usa en el JMP FAR.
Crucialmente, la TSS contiene ESP0 y SS0, que determinan qué pila usar cuando hay un cambio de privilegio (nivel 3 → 0). Esto permite que cada tarea tenga su propia pila de kernel, aislada de su pila de usuario.
IDT - Interrupt Descriptor Table
La IDT es una tabla de 256 entradas que mapea cada número de interrupción/excepción a su handler. Cada entrada (interrupt gate o trap gate) especifica: selector de código (normalmente GDT_CODE_0_SEL), offset dentro de ese segmento (dirección del handler), DPL (qué nivel puede invocarlo), y tipo (interrupt gate vs trap gate).
Interrupt gates deshabilitan interrupciones al entrar (clear IF). Trap gates las dejan habilitadas. Para la mayoría de handlers, queremos interrupt gates para evitar que otra interrupción ocurra en medio.
El DPL es crítico: DPL=0 significa "solo nivel 0 puede invocar", DPL=3 significa "cualquier nivel puede invocar". Syscalls usan DPL=3, excepciones/IRQs usan DPL=0.

## FLUJOS TÍPICOS
Syscall Completa

Usuario: Tarea en nivel 3 configura parámetros en registros (EDI, ESI, ECX, etc.)
Usuario: Ejecuta INT 0x90 (por ejemplo)
CPU: Busca entrada 0x90 en IDT, encuentra DPL=3 (permitido)
CPU: Cambia a nivel 0, busca ESP0/SS0 en TSS, cambia de stack
CPU: Pushea en stack de kernel: SS usuario, ESP usuario, EFLAGS, CS usuario, EIP usuario
CPU: Salta a ISR_90 en nivel 0
ISR (asm): Pushad (guarda registros)
ISR (asm): Pushea parámetros específicos de los registros a la pila
ISR (asm): Call a función C del kernel
