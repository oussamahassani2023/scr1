# Лабораторная работа № 2 — SCR1
## Вариант 10: обработка исключения Breakpoint

**Студент:** Хассани Уссама (Hassani Oussama), группа P4119, Университет ИТМО

### 1. Цель работы

Ознакомиться с возможностями настройки архитектуры системы команд, обработкой исключений и принципами работы сборщика программ (linker) на примере процессорного ядра SCR1.

### 2. Задание (вариант 10)

| Параметр | Значение |
|---|---|
| Вид исключения | Breakpoint |
| Тест | `isa/rv32mi/sbreak.S` |
| Reset Vector | `0x400` |
| Trap Vector | `0xA00` |
| Обработчик | вывод строки `breakbreak` |

Требовалось:

1. Создать копию (fork) репозитория SCR1 и ветку `lab_scr1_sim`.
2. Убедиться, что до изменений проект собирается и все тесты проходят.
3. Оставить в списке тестов только тест варианта.
4. Изменить обработку исключений `trap_vector` в `riscv_macros.h`.
5. Задать Reset Vector и Trap Vector в `scr1_arch_description.svh`.
6. Изменить linker-скрипт `link.ld` под новые значения векторов.
7. Сохранить результаты моделирования, дизассемблер и трейс-логи в `results`.

### 3. Выполненные изменения

| Файл | Изменение |
|---|---|
| `sim/tests/riscv_isa/rv32_tests.inc` | в списке тестов оставлен только `isa/rv32mi/sbreak.S` |
| `src/includes/scr1_arch_description.svh` | `SCR1_ARCH_RST_VECTOR = 'h400`, `SCR1_ARCH_MTVEC_BASE = 'hA00` |
| `sim/tests/common/link.ld` | код `.text.init` (точка входа `_start`) размещён с адреса `0x400`; вектор исключений вынесен в отдельную секцию `.trap_vector`, размещённую по адресу `0xA00` |
| `sim/tests/common/riscv_macros.h` | `trap_vector` помещён в секцию `.trap_vector`, `_start` — в `.text.init`; в обработчик добавлен вывод строки при исключении Breakpoint |
| `src/tb/scr1_top_tb_ahb.sv`, `src/tb/scr1_top_tb_axi.sv` | адреса `ADDR_START`, `ADDR_TRAP_VECTOR`, `ADDR_TRAP_DEFAULT` приведены к новым значениям (используются при проверке тестов compliance/arch) |

#### 3.1. Параметры ядра

```systemverilog
parameter bit [`SCR1_XLEN-1:0] SCR1_ARCH_RST_VECTOR = 'h400;  // адрес первой команды после сброса
parameter bit [`SCR1_XLEN-1:0] SCR1_ARCH_MTVEC_BASE = 'hA00;  // начальное значение mtvec.base
```

В конфигурации MAX записываются только старшие 26 бит поля `mtvec.base` (`SCR1_MTVEC_BASE_WR_BITS = 26`), поэтому адрес вектора должен быть выровнен на 64 байта. Значение `0xA00` этому условию удовлетворяет.

#### 3.2. Linker-скрипт

```ld
.text.init 0 : {
    FILL(0);
    . = 0x100 - 12;
    SIM_EXIT = .;        /* 0xF4 */
    LONG(0x13);
    SIM_STOP = .;        /* 0xF8: бесконечный цикл, адрес завершения моделирования */
    LONG(0x6F);
    LONG(-1);
    . = 0x400;           /* Reset Vector */
    PROVIDE(__TEXT_START__ = .);
    *(.text.init)
} >RAM

.trap_vector 0xA00 : {   /* Trap Vector */
    KEEP(*(.trap_vector))
} >RAM
```

Адреса `SIM_EXIT` и `SIM_STOP` оставлены прежними: тестовое окружение завершает моделирование, когда `pc` достигает `0xF8`.

Важная особенность: если у секции указана область памяти (`>RAM`), а адрес не задан явно, компоновщик размещает её сразу за предыдущей секцией региона и **игнорирует** предшествующее присваивание `. = 0xA00;`. Поэтому для секции `.trap_vector` адрес `0xA00` указан явно.

#### 3.3. Обработчик исключений

В макросе `RVTEST_CODE_BEGIN` код вектора исключений помещён в отдельную секцию, а точка входа — в `.text.init`:

```asm
.section .trap_vector,"ax"
.balign 4
trap_vector:
    csrr a4, mcause
    ... (проверка ecall из U/S/M-режима -> _report)
    /* вывод breakbreak для исключения Breakpoint */
    csrr a4, mcause
    li   a5, CAUSE_BREAKPOINT        # 3
    bne  a4, a5, 2f
    li   t0, 0xf0000000              # порт вывода тестового окружения
    li   t1, 'b'; sb t1, 0(t0)
    li   t1, 'r'; sb t1, 0(t0)
    ... (всего 10 символов: b r e a k b r e a k)
2:  la   a4, mtvec_handler           # продолжение штатной обработки
    beqz a4, 1f
    jr   a4
    ...
_report:
    j sc_exit

.section .text.init,"ax"
.balign 64
.globl _start
_start:
    ...
```

Каждый байт, записанный по адресу `0xF0000000`, тестовое окружение выводит на консоль. После вывода обработчик передаёт управление штатному `mtvec_handler` теста, поэтому проверка теста выполняется как обычно.

### 4. Ход выполнения теста

1. Ядро стартует с `0x400` (`_start`), записывает в `mtvec` адрес `trap_vector` и через `mret` переходит к коду теста (`_run_test`).
2. Выполняется `ebreak`: в `mepc` сохраняется адрес команды, в `mcause` записывается `3`, управление передаётся на `0xA00`.
3. `trap_vector` определяет причину (`mcause = 3`), выводит `breakbreak` и переходит к `mtvec_handler` теста.
4. `mtvec_handler` проверяет `mcause` и `mepc` и переходит на метку `pass`.
5. `pass` выполняет `ecall` (`mcause = 11`): `trap_vector` направляет его в `_report` → `sc_exit` → `SIM_EXIT` (`0xF4`) → `SIM_STOP` (`0xF8`).
6. Тестовое окружение фиксирует `pc = 0xF8`, проверяет `a0 = 0` и считает тест пройденным.

### 5. Сборка и запуск

Окружение: Verilator 5.052, RISC-V GNU toolchain `riscv64-unknown-elf-gcc 14.2.0`, конфигурация `AHB_MAX_imc`.

```bash
make clean
make run_verilator TARGETS=riscv_isa
```

Вывод:

```text
---Test:                       sbreak.hex
breakbreakTest passed

# Summary: 1/1 tests passed

                          Test               | build | simulation
                      sbreak.hex                OK        PASS
```

![Рис. 1 — результат запуска `make run_verilator TARGETS=riscv_isa`: выведено `breakbreak`, `1/1 tests passed`](images/01_run_verilator_pass.jpg)

*Рис. 1 — результат моделирования: тест `sbreak` пройден, обработчик вывел `breakbreak`.*

Адреса в дизассемблере (`results/sbreak.dump`):

```text
00000400 <_start>:
00000a00 <trap_vector>:
```

Они совпадают с заданными значениями Reset Vector и Trap Vector.

![Рис. 2 — копирование результатов в `results` и проверка адресов `_start` и `trap_vector` в дизассемблере](images/02_dump_addresses.jpg)

*Рис. 2 — адреса `_start = 0x400` и `trap_vector = 0xA00` в файле `results/sbreak.dump`.*

Запуск в режиме формирования волновых форм и трейс-логов (`make run_verilator_wf TARGETS=riscv_isa TRACE=1`) также завершился успешно: `1/1 tests passed`.

![Рис. 3 — результат запуска в режиме `run_verilator_wf` с `TRACE=1`](images/03_run_verilator_wf_pass.jpg)

*Рис. 3 — запуск с формированием волновых форм и трейс-логов (каталог сборки `verilator_wf_..._TRACE_1`): тест пройден.*

### 6. Результаты (каталог `results`)

- `test_results.txt` — результат моделирования (`sbreak.hex  OK  PASS`);
- `sim_results.txt` — полный вывод симулятора;
- `sbreak.dump` — дизассемблер теста;
- трейс-логи регистрового файла (MPRF) и CSR, полученные при запуске с `TRACE=1`.

### 7. Возникшие проблемы и их решение

Первый запуск после внесения изменений завершился ошибкой `TIMEOUT` (0/1 tests passed), хотя исходная конфигурация проходила все 56 тестов. Были найдены три причины.

1. **Устаревший ELF-файл.** В Makefile теста `riscv_isa` зависимости заданы так, что `.o` и `.elf` пересобираются только при изменении исходного `.S`-файла, но не при изменении `riscv_macros.h` и `link.ld`. RTL при этом пересобирался с новым Reset Vector (`0x400`), а программа в памяти оставалась по старым адресам (`_start = 0x200`, `trap_vector = 0x1C0`). Ядро стартовало с `0x400`, исполняло пустую память и не доходило до `0xF8`. Решение: перед каждым запуском выполнять `make clean`.
2. **Неверное размещение секции `.trap_vector`.** Запись `. = 0xA00;` перед секцией с `>RAM` не действует (см. раздел 3.2), и секция оказывалась сразу за `.text.init`. Решение: явно указать адрес в заголовке секции — `.trap_vector 0xA00 : { ... } >RAM`. Поведение проверено на GNU ld 2.42.
3. **Пустые файлы в одном из коммитов.** При фиксации изменений пять файлов (`riscv_macros.h`, `scr1_arch_description.svh`, `rv32_tests.inc`, `scr1_top_tb_ahb.sv`, `scr1_top_tb_axi.sv`) были записаны пустыми. Файлы восстановлены из предыдущего коммита командой `git checkout <commit> -- <файлы>`.

### 8. Вывод

В ходе работы изучены настройка векторов сброса и обработки исключений ядра SCR1, обработка исключения Breakpoint (`ebreak`, `mcause = 3`, `mepc`, `mtvec`) и роль linker-скрипта при размещении кода по заданным адресам. Для варианта 10 заданы `Reset Vector = 0x400` и `Trap Vector = 0xA00`, обработчик выводит строку `breakbreak`. Тест `isa/rv32mi/sbreak.S` проходит успешно (`1/1 tests passed`), адреса `_start` и `trap_vector` в дизассемблере совпадают с заданными.

Отдельно получен практический опыт отладки: расхождение между пересобираемым RTL и устаревшим ELF приводит к тайм-ауту, поэтому между запусками необходимо выполнять `make clean`.
