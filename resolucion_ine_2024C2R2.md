Recu Segundo Parcial
Segundo Cuatrimestre 2024

#include "mmu.h"
#include "i386.h"
#include "defines.h"
#include "tss.h"
#include "kassert.h"
#include "gdt.h"
#include "gdt.c"
#include "idt.h"
#include "idt.c"
#include "isr.h"
#include "mmu.c"
#include "sched.h"
#include "sched.c"
#include "shared.h"
#include "tss.c"
#include "types.h"

void idt_init() {
  ...
  IDT_ENTRY3(88);
  IDT_ENTRY3(98);

  agrego una entrada para la syscall
  IDT_ENTRY3(99);
}

global _isr99
_isr99:
pushad

push edi ; indice tarea destino

call swap

add esp, 4

popad
iret

typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  TASK_BLOCKED //agrego
} task_state_t;

typedef struct {
  int16_t selector;
  task_state_t state;
  int8_t swap; //inicializa en -1 entonces si no esta esperando ningun swap esta en -1
  bool llamo_swap_now_en_ultima_ronda //inicializa en 0
} sched_entry_t;

void swap(int8_t tarea_destino){
    if(llamo_swap_con(tarea_destino, current_task)){
        intercambiar_registros(tarea_destino, current_task);
        se_realizo_intercambio(tarea_destino);
        se_realizo_intercambio(current_task);
        sched_tasks[tarea_destino].state = TASK_RUNNABLE;
        sched_tasks[tarea_destino].swap = -1;
    }
    else{
        no_se_realizo_intercambio(tarea_destino);
        no_se_realizo_intercambio(current_task);
        sched_tasks[current_task].swap = tarea_destino;
        sched_tasks[current_task].state = TASK_BLOCKED;
        cambiar_tarea();
    }
}

cambiar_tarea:
  pushad 

  call sched_next_task

  mov word [sched_task_selector], ax
  jmp far [sched_task_offset]

  popad
  ret

int8_t llamo_swap_con(int8_t tarea_a_chequear int8_t tarea_destino){
    return (sched_tasks[tarea_a_chequear].swap == tarea_destino)
}

void intercambiar_registros(int8_t tarea_destino, int8_t curent_task){
    uint32_t* pila = tss_tasks[current_task]->esp;

    intercambiar_registro(&pila[7], &tss_tasks[tarea2].eax);
    intercambiar_registro(&pila[4], &tss_tasks[tarea2].ebx);
    intercambiar_registro(&pila[6], &tss_tasks[tarea2].ecx);
    intercambiar_registro(&pila[5], &tss_tasks[tarea2].edx);
    intercambiar_registro(&pila[0], &tss_tasks[tarea2].edi);
    intercambiar_registro(&pila[1], &tss_tasks[tarea2].esi);
}

void intercambiar_registro(uint32_t* registro1, uint32_t* registro2){
    uint32_t tmp = *registro1;   // Guardo el valor al que apunta registro1
    *registro1 = *registro2;     // Copio el valor de registro2 en registro1
    *registro2 = tmp;            // Copio el valor original de registro1 en registro2
}

void idt_init() {
  ...
  IDT_ENTRY3(88);
  IDT_ENTRY3(98);

  agrego una entrada para la syscall
  IDT_ENTRY3(99);
  IDT_ENTRY3(100)
}

global _isr100
_isr100:
pushad

push edi ; indice tarea destino

call swap_now

add esp, 4

popad
iret

void swap_now(int8_t tarea_destino){
    if(llamo_swap_con(tarea_destino, current_task)){
        intercambiar_registros(tarea_destino, current_task);
        se_realizo_intercambio(tarea_destino);
        se_realizo_intercambio(current_task);
        if(sched_tasks[tarea_destino].llamo_swap_now_ultima_ronda){
            sched_tasks[tarea_destino].llamo_swap_now_ultima_ronda = 0;
        }
        sched_tasks[tarea_destino].state = TASK_RUNNABLE;
        sched_tasks[tarea_destino].swap = -1;
    }
    else{
        no_se_realizo_intercambio(tarea_destino);
        no_se_realizo_intercambio(current_task);
        sched_tasks[current_task].swap = tarea_destino;
        sched_tasks[current_task].llamo_swap_now_ultima_ronda = 1;
        cambiar_tarea();
    }
}

uint16_t sched_next_task(void) {
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
    if(sched_tasks[i].llamo_swap_now_ultima_ronda){
        sched_tasks[i].llamo_swap_now_ultima_ronda = 0;
        sched_tasks[i].swap = -1;
    }
    return sched_tasks[i].selector;
  }

  // En el peor de los casos no hay ninguna tarea viva. Usemos la idle como
  // selector.
  return GDT_IDX_TASK_IDLE << 3;
}

void se_realizo_intercambio(int8_t tarea){
    uint32_t cr3 = tss_tasks[tarea].cr3;
    paddr_t direc_fisica = virt_to_phy(cr3, (vaddr_t)0xC001C0DE);

    mmu_map_page(rcr3(), SRC_VIRT_PAGE, direc_fisica, MMU_P | MMU_W | MMU_S); //se la mapeo al kernel xq como estoy en la syscall el kernel va a escribir el 1 o 0

    *(uint32_t*)SRC_VIRT_PAGE = 1;

    mmu_unmap_page(rcr3(), SRC_VIRT_PAGE);
}

void no_se_realizo_intercambio(int8_t tarea){
    uint32_t cr3 = tss_tasks[tarea].cr3;
    paddr_t direc_fisica = virt_to_phy(cr3, (vaddr_t)0xC001C0DE);

    mmu_map_page(rcr3(), SRC_VIRT_PAGE, direc_fisica, MMU_P | MMU_W | MMU_S); //se la mapeo al kernel xq como estoy en la syscall el kernel va a escribir el 1 o 0

    *(uint32_t*)SRC_VIRT_PAGE = 0;

    mmu_unmap_page(rcr3(), SRC_VIRT_PAGE);
}

paddr_t virt_to_phy(uint32_t cr3, vaddr_t virt) {

    uint32_t* cr3 = task_selector_to_cr3(task_id);
    uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);
    uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;
    return (paddr_t) (pt[pt_index].page << 12);
}
