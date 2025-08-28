### Программа 1

```asm
global _main
extern _printf

section .data
    array    dq 3, 17, 5, 42, 8, 15
    arr_len  equ ($ - array) / 8
    fmt      db "Максимальное значение: %d", 10, 0

section .bss
    max_num  resq 1

section .text
_main:
    lea rbx, [rel array]    
    mov rcx, arr_len
    mov rax, [rbx]            
    mov [rel max_num], rax    
    add rbx, 8
    dec rcx

.loop:
    cmp rcx, 0
    je .done
    mov rax, [rbx]
    cmp rax, [rel max_num]
    jle .next
    mov [rel max_num], rax

.next:
    add rbx, 8
    dec rcx
    jmp .loop

.done:
    lea rdi, [rel fmt]
    mov rsi, [rel max_num]
    mov rax, 0
    call _printf

    mov rax, 0
    ret
```

### Программа 2

```asm
global _start

extern _printf
extern _srand
extern _rand
extern _malloc
extern _free
extern _sqrt
extern _pthread_create
extern _pthread_join
extern _time
extern _exit

SYS_EXIT equ 1

%macro inc_cnt 1
    xor rax, rax
    xor rdi, rdi
    mov rax, %1
    mov rdi, [rax]
    inc rdi
    mov [rax], rdi
%endmacro

%macro print 1
    lea rdi, [%1]
    xor rax, rax
    call _printf
%endmacro

%macro print_two 2
    lea rdi, [%1]
    mov rsi, %2
    xor rax, rax
    call _printf
%endmacro

%macro exit 0
    mov rax, SYS_EXIT
    xor rdi, rdi
    syscall
%endmacro

section .bss
    MEMORY_LOC resq 1
    SUM_1_ARRAY resq 64
    SUM_2_ARRAY resq 64
    THREAD_ID_ARRAY resq 64
    NUMBERBUFFER resq 64

section .data
    NEWLINE db 10, 0
    MSG_ERROR_MEMORY db "Could not allocate memory", 10, 0
    MSG_ERROR_PTHREAD db "Thread creation error :  %d", 10, 0
    MSG_ALERT_NO_ARG db "No Arguments, defaults values set", 10, 0
    MSG_ALERT_NO_LINES db "No Lines arg, default values set", 10, 0
    MSG_NUMBER_THREADS db "Number of threads : %ld",10 , 0
    MSG_NUMBER_LINES db "Number of Lines : %ld",10 , 0
    MSG_RANDOM_CONST db "Random Seed Value : %lf",10 , 0
    MSG_RANDOM_TIME db "Random Seed Value set for time.",10 , 0
    MSG_SUMMARY_TITLE db "-----Summary-----", 10, 0
    MSG_SUMMARY_HEADER db "-----------------", 10, 0
    MSG_SUMMARY_AVG db "Average : %lf",10 , 0
    MSG_SUMMARY_DEV db "Deviation : %lf",10 , 0

    N_THREADS dq 4
    SIZE_ARRAY dq 100000
    THREAD_DUTY dq 0
    RAND_SEED dq 0
    THREAD_CNT dq 0
    SUM_1 dq 0
    SUM_2 dq 0
    MEMORY_POINTER dq 0
    MAX_VAL dq 0x7fffffff

section .text

_start:
    push rbp
    mov rbp, rsp

    mov r13, [rbp + 16]
    mov r14, [rbp + 24] 
    mov r15, [rbp + 32] 

    call set_values
    call set_thread_duty
    call memory_alloc
    call create_thread
    call join_thread
    call memory_free

    print MSG_SUMMARY_HEADER
    print MSG_SUMMARY_TITLE
    print MSG_SUMMARY_HEADER

    call get_avg
    call get_dev

    pop rbp
    exit

memory_fill:
    push rbp
    mov rbp, rsp
    mov r15, rdi
    mov rax, [THREAD_DUTY]
    add rax, rdi
    mov r14, rdi

    cmp rdi, 0
    je first_thread

    mov rdx, 0
    mov rax, r14
    mov r12, [THREAD_DUTY]
    div r12
    jmp other_threads

first_thread:
    mov rax, 0
    jmp other_threads

other_threads:
    mov r13, rax
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, SUM_1_ARRAY
    add rax, r10
    mov r12, rax

    mov rax, r13
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, SUM_2_ARRAY
    add rax, r10
    mov r13, rax

    mov rax, r14
    mov rdi, [THREAD_DUTY]
    add rax, rdi
    mov r14, rax

memory_fill_lp:
    xor rax, rax
    call _rand
    CVTSI2SD xmm0, rax
    CVTSI2SD xmm1, [MAX_VAL]
    divsd xmm0, xmm1

    mov rax, r15
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, [MEMORY_LOC]
    add rax, r10
    movlps qword [rax], xmm0

    movq xmm1, xmm0
    movq xmm2, xmm0
    movq xmm1, [r12]
    addsd xmm0, xmm1
    movq [r12], xmm0

    pxor xmm0, xmm0
    mulsd xmm2, xmm2
    movq xmm0, [r13]
    addsd xmm0, xmm2
    movq [r13], xmm0
    pxor xmm0, xmm0

    inc r15
    cmp r15, r14
    jb memory_fill_lp

    pop rbp
    ret

set_thread_duty:
    xor rdx, rdx
    mov rax, [SIZE_ARRAY]
    mov r10, [N_THREADS]
    div r10
    mov [THREAD_DUTY], rax
    ret

set_values:
    push rbp
    mov rdx, 0
    cmp rdx, r13
    je no_arg

    mov dword [NUMBERBUFFER], 0
    mov [NUMBERBUFFER], r13
    call conv_int
    mov [N_THREADS], rax

    cmp r14, 0
    je no_lines_arg

    mov [NUMBERBUFFER], r14
    call conv_int
    mov [SIZE_ARRAY], rax

    cmp r15, 0
    je print_start

    mov [NUMBERBUFFER], r15
    call conv_flt
    jmp print_start

no_lines_arg:
    print MSG_ALERT_NO_LINES
    jmp print_start

no_arg:
    print MSG_ALERT_NO_ARG

print_start:
    print_two MSG_NUMBER_THREADS, [N_THREADS]
    print_two MSG_NUMBER_LINES, [SIZE_ARRAY]

    mov rax, [RAND_SEED]
    cmp rax, 0
    je ran_t
    call seed_random_const
    jmp exit_val

ran_t:
    call seed_random_time

exit_val:
    pop rbp
    ret

get_avg:
    push rbp
    movlps xmm0, [SUM_1]
    CVTSI2SD xmm1, [SIZE_ARRAY]
    divsd xmm0, xmm1
    movlps [SUM_1], xmm0

    mov rax, 1
    lea rdi, [MSG_SUMMARY_AVG]
    xor rsi, rsi
    call _printf
    pop rbp
    ret

get_dev:
    push rbp
    movlps xmm0, [SUM_2]
    CVTSI2SD xmm1, [SIZE_ARRAY]
    divsd xmm0, xmm1
    movlps xmm3, [SUM_1]
    mulsd xmm3, xmm3
    subsd xmm0, xmm3
    call _sqrt

    mov rax, 1
    lea rdi, [MSG_SUMMARY_DEV]
    xor rsi, rsi
    call _printf
    pop rbp
    ret

memory_free:
    mov rdi, [MEMORY_LOC]
    call _free
    ret

memory_alloc:
    xor rax, rax
    mov rdi, [SIZE_ARRAY]
    imul rdi, 8
    call _malloc
    cmp rax, 0
    je error_memory
    mov [MEMORY_LOC], rax
    ret

join_thread:
    push rbp
    xor rax, rax
    mov [THREAD_CNT], rax

join_thread_lp:
    mov rax, [THREAD_CNT]
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, THREAD_ID_ARRAY
    add rax, r10

    xor rsi, rsi
    mov rdi, [rax]
    call _pthread_join

    mov rax, [THREAD_CNT]
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, SUM_1_ARRAY
    add rax, r10
    mov r13, rax

    movlps xmm0, [r13]
    movlps xmm1, [SUM_1]
    addsd xmm0, xmm1
    movq [SUM_1], xmm0
    pxor xmm0, xmm0

    mov rax, [THREAD_CNT]
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, SUM_2_ARRAY
    add rax, r10
    mov r13, rax

    movlps xmm0, [r13]
    movlps xmm1, [SUM_2]
    addsd xmm0, xmm1
    movq [SUM_2], xmm0
    pxor xmm0, xmm0

    inc_cnt THREAD_CNT
    mov r10, [THREAD_CNT]
    cmp r10, [N_THREADS]
    jne join_thread_lp
    pop rbp
    ret

create_thread:
    push rbp
    xor rax, rax
    mov [THREAD_CNT], rax

create_thread_lp:
    mov rax, [THREAD_DUTY]
    mov r10, [THREAD_CNT]
    mul r10
    mov r11, rax

    mov rax, [THREAD_CNT]
    mov r10, 8
    mul r10
    mov r10, rax
    mov rax, THREAD_ID_ARRAY
    add rax, r10

    mov rdi, rax
    xor rsi, rsi
    lea rdx, memory_fill
    mov rcx, r11
    call _pthread_create

    cmp rax, 0
    jne error_thread

    inc_cnt THREAD_CNT
    mov r10, [THREAD_CNT]
    cmp r10, [N_THREADS]
    jb create_thread_lp
    pop rbp
    ret

conv_int:
    lea rdi, [NUMBERBUFFER]
    call _atol
    ret

conv_flt:
    lea rdi, [NUMBERBUFFER]
    call _atof
    movlps [RAND_SEED], xmm0
    ret

seed_random_const:
    push rbp
    movlps xmm0, [RAND_SEED]
    mov rax, 1
    lea rdi, [MSG_RANDOM_CONST]
    xor rsi, rsi
    call _printf
    pop rbp
    mov rdi, [RAND_SEED]
    call _srand
    ret

seed_random_time:
    print MSG_RANDOM_TIME
    xor rax, rax
    xor rdi, rdi
    call _time
    mov rdi, rax
    call _srand
    ret

error_memory:
    lea rdi, [MSG_ERROR_MEMORY]
    xor rax, rax
    call _printf
    exit

error_thread:
    mov rsi, rax
    lea rdi, [MSG_ERROR_PTHREAD]
    xor rax, rax
    call _printf
    exit
```