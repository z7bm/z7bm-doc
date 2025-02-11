# Загрузчик

## Общие сведения и назначение

Загрузчик предназначен для выполнения операций начальной инициализации аппаратуры SoC, конфигурирования памяти (таблица трансляции адресов **MMU** и размещение сегментов **OCM**) и загрузки образа прикладной программы из флешь памяти в **OCM** с последующей передачей ей управления.

На начальном этапе загрузчик производит низкоуровневую инициализацию (**SCU**, **MMU**, кэши **L1** и **L2** уровней), настройку режимов процессора (**SPSR** и **SP**) и выполняет код стартапа **C/C++**.

Затем управление передаётся функции `main` загрузчика, которая осуществляет ремап таблицы трансляции **MMU**, перемещение сегментов **OCM** и собственно загрузку образа прикладной программы из флешь памяти в **OCM**.

Начальный код загрузчика реализован на основе **FSBL** (First Stage Boot Loader) для **Zynq-7000**, который в свою очередь во многом базируется на референсном коде от **ARM**. Ниже приведено конспективное описание работы загрузчика.

## Старт

Работа программы начинается с кода, определённого в файле `boot.S`.

По нулевым адресам расположена таблица векторов прерываний (исключений):

```asm
    B   _boot
    B   Undefined
    B   SVCHandler
    B   PrefetchAbortHandler
    B   DataAbortHandler
    NOP /* Placeholder for address exception vector*/
    B   IRQHandler
    B   FIQHandler
```

Старт программы происходит по вектору `_boot`, где расположен низкоуровневый код инициализации аппаратуры процессора



## Низкоуровневая инициализация

### Настройка APU

Сначала производится ряд действий, непосредственно связанных я ядрами SoC.

Первое действие - проверка идентификатора `CPU`, продолжать работу может только `CPU0`, остальные должны встать на ожидание:

```asm
        /* only allow cpu0 through */
    mrc  p15,0,r1,c0,c0,5
    and  r1, r1, #0xf
    cmp  r1, #0
    beq  CheckEFUSE
EndlessLoop0:
    wfe
    b   EndlessLoop0
```
Здесь считывается Affinity Register процессора (расположенный в сопроцессоре CP15), в младших битах которого закодирован идентификатор (номер) ядра, далее проверяется значение этого поля, если оно не ноль, то ядро паркуется  в `wfe`.

Далее проверяется, сколько ядер у `APU`, если одно, то второе ядро сбрасывается. Два непонятных момента:

1. Информация о количестве ядер считывается из некоего регистра `EFUSEStatus`, описание которого в документации найти не удалось. 
1. Если ядро `CPU1` отсутствует, то не ясно, зачем держать его в сбросе и при останавливать клок, а именно это в коде и делается (`orr r1, r1, #0x22` - установка битов сброса и останова клока для `CPU1`):

```asm
    ldr r0,=SLCRCPURSTReg
    ldr r1,[r0]                             /* Read CPU Software Reset Control register */
    orr r1,r1,#0x22
    str r1,[r0]                             /* Reset CPU1 */
```
Возможно, это связано с тем, что хотя самого ядра и нет, но окружающая его инфраструктура  никуда не делась (она является частью общей системы), и чтобы она не потребляла и/или не создавала каких-либо других проблем, её нужно отключить. Это предположение, в документации никаких сведений об этом обнаружить не удалось.

Затем производится извлечение номера ревизии процессора и обрабатываются две errata. Что именно тут делается, не ясно, производится модификация битов какого-то диагностического регистра (а может, и не производится - операции условные). Всё это не документировано. 


### Таблица векторов прерываний

Выполняется установка таблицы векторов прерываний:

```asm
    ldr r0, =vector_base
    mcr p15, 0, r0, c12, c0, 0
```

`CP15->c12`&nbsp;– это регистр `VBAR`, т.е. как раз и есть Vector Base Address Register.

### Инвалидация SCU, MMU, L1 Caches

Производится инвалидация кэшей, таблицы **MMU** и предсказателя ветвлений. 

#### SCU Invalidate All Rregisters in Secure State Register

Инвалидация всех каналов (ways) на уровне SCU.

```asm
    /*invalidate scu*/
    ldr r7, =0xf8f0000c
    ldr r6, =0xffff
    str r6, [r7]
```

#### Инвалидация TLB, кэша инструкций, предсказателя ветвлений

```asm
    /* Invalidate caches and TLBs */
    mov r0,#0                       /* r0 = 0  */
    mcr p15, 0, r0, c8, c7, 0       /* invalidate TLBs */
    mcr p15, 0, r0, c7, c5, 0       /* invalidate icache */
    mcr p15, 0, r0, c7, c5, 6       /* Invalidate branch predictor array */
    bl  invalidate_dcache           /* invalidate dcache */
```

#### Инвалидация кэша данных

Подпрограмма `invalidate_dcache` выполняет инвалидацию кэша данных. Процессор Cortex-A9 не имеет инструкции, которая могла бы инвалидировать весь кэш данных, поэтому приходится инвалидировать по одной линии.

Первым делом определяется уровень кэша:

```asm
   mrc  p15, 1, r0, c0, c0, 1   /* read CLIDR */
   ands r3, r0, #0x7000000
   mov  r3, r3, lsr #23         /* cache level value (naturally aligned) */
   beq  finished
```
Здесь из регистра `CLIDR` (Cache Level ID Register) сопроцессора `CP15` считывается значение, маскируется поле `LoUU` (Level of unification, uniprocessor, 29:27 биты), результат "нормализуется" - поле перемещается в младшие биты, после чего в результирующем регистре содержится уровень кэша (в нашем случае это значение равно 2).
Если уровень кэша 0, то инвалидация не производится и процессор переходит к выполнению эпилога подпрограммы.


!!! tip "**ARMv7 Architecture Reference Manual**"

    LoUU, Level of unification, uniprocessor This field defines the last level of cache that must be cleaned or invalidated when cleaning or invalidating to the point of unification for the processor. As with LoC, the LoUU value is a cache level. If the LoUU field value is 0x0, this means that no levels of cache need to cleaned or invalidated when cleaning or invalidating to the point of unification. If the LoUU field value is a nonzero value that corresponds to a level that is not implemented, this indicates that all implemented caches are before the point of unification.

Затем, собственно, производится инвалидация кэша данных и возврат к продолжению инициализации:

```asm
    mov r10, #0             /* start with level 0 */
loop1:
    add r2, r10, r10, lsr #1       /* work out 3xcachelevel */
    mov r1, r0, lsr r2             /* bottom 3 bits are the Cache type for this level */
    and r1, r1, #7                 /* get those 3 bits alone */
    cmp r1, #2
    blt skip                       /* no cache or only instruction cache at this level */
    mcr p15, 2, r10, c0, c0, 0     /* write the Cache Size selection register */
    isb                            /* isb to sync the change to the CacheSizeID reg */
    mrc p15, 1, r1, c0, c0, 0      /* reads current Cache Size ID register */
    and r2, r1, #7                 /* extract the line length field */
    add r2, r2, #4                 /* add 4 for the line length offset (log2 16 bytes) */
    ldr r4, =0x3ff
    ands    r4, r4, r1, lsr #3     /* r4 is the max number on the way size (right aligned) */
    clz r5, r4                     /* r5 is the bit position of the way size increment */
    ldr r7, =0x7fff
    ands    r7, r7, r1, lsr #13    /* r7 is the max number of the index size (right aligned) */
loop2:
    mov r9, r4                     /* r9 working copy of the max way size (right aligned) */
loop3:
    orr r11, r10, r9, lsl r5       /* factor in the way number and cache number into r11 */
    orr r11, r11, r7, lsl r2       /* factor in the index number */
    mcr p15, 0, r11, c7, c6, 2     /* invalidate by set/way */
    subs    r9, r9, #1             /* decrement the way number */
    bge loop3   
    subs    r7, r7, #1             /* decrement the index */
    bge loop2   
skip:   
    add r10, r10, #2               /* increment the cache number */
    cmp r3, r10   
    bgt loop1

finished:
    mov r10, #0                    /* swith back to cache level 0 */
    mcr p15, 2, r10, c0, c0, 0     /* select current cache level in cssr */
    dsb
    isb

    bx  lr
```

Этот код вычисляет уровень кэша и производит в цикле инвалидацию кэша по схеме "way-line", т.е. проходит вложенный цикл (первый уровень - проход по ways, второй - проход по линиям). Для L1 кэша это 4 цикла по way и 256 циклов по линиям.

Фрагмент с `loop1` до `loop2` - обработка сведений о структуре кэшей, извлекаются параметры кэша текущего уровня (сколько way, сколько линий и т.д.). Фрагмент с `loop2` до `skip` выполняет собственно инвалидацию. Внешний цикл (`loop2`-`skip`) обрабатывает ways кэша, вложенный цикл (`loop3`-`bge loop3`) непосредственно инвалидирует линии кэша.

Код рассчитан на инвалидацию обоих уровней (L1 и L2), но реально при загрузке при первом вызове обрабатывается только L1 кэш, L2 неактивен - видимо потому, что он (L2  кэш) ещё не разрешён.

В конце производится переключение регистра, определяющего текущий уровень кэша, на значение, соответствующее уровню L1 (это актуально, если подпрограмма обрабатывала кэши разных уровней).

### Запрещение MMU
```asm
    /* Disable MMU, if enabled */
    mrc p15, 0, r0, c1, c0, 0      /* read CP15 register 1 */
    bic r0, r0, #0x1               /* clear bit 0 */
    mcr p15, 0, r0, c1, c0, 0      /* write value back */
```

Хотя **MMU** в этот момент и так запрещено. Запрет производится через сброс бита `M` в регистре `SCTLR`, бит этот называется в документации от ARM: `MPU` enable bit. В `Cortex-A` нет `MPU`, а вместо него **MMU**. Имеется ли в виду, что подразумевается устройство в зависимости от контекста, не ясно. По коду видимо так.

### Настройка SPSR и SP

Производится настройка `SPSR` и `SP` для режимов `IRQ`, `SVC`, `Abort`, `FIQ`, `Undefined`, `System (User)`. Код по сути один и тот же, меняется только режим (путём установки битового поля `M` в `CPSR`). Код для `IRQ`:

```asm
    mrs r0, cpsr               /* get the current PSR */
    mvn r1, #0x1f              /* set up the irq stack pointer */
    and r2, r1, r0
    orr r2, r2, #0x12          /* IRQ mode */
    msr cpsr, r2
    ldr r13,=IRQ_stack         /* IRQ stack pointer */
    bic r2, r2, #(0x1 << 9)    /* Set EE bit to little-endian */
    msr spsr_fsxc,r2
```

Константа в инструкции `orr r2, r2, #0x12` определяет режим, следующая за ней `msr cpsr, r2` меняет режим. 
`ldr r13,=IRQ_stack` загружает значение указателя стека в `SP`, а `msr spsr_fsxc,r2` инициализирует статусный регистр этого режима.

Обработка остальных режимов, перечисленных выше, точно такая же, меняются только константы режимов и адреса стеков.

### Включение SCU, MMU и кэшей L1

После настройки режимов производится включение `SCU`, **MMU** и кэшей первого уровня.

#### Включение SCU

```asm
    /*set scu enable bit in scu*/
    ldr r7, =0xf8f00000
    ldr r0, [r7]
    orr r0, r0, #0x1
    str r0, [r7]
```

Здесь всё просто: `0xf8f00000` - это адрес регистра `SCU_CONTROL_REGISTER` из группы `mpcore`, младший бит этого регистра `SCU_enable`. Вышеприведённый код просто устанавливает этот бит, разрешая тем самым работу `SCU`.

#### MMU

Затем производится настройка регистра `TTBR0` из группы управления **MMU**:

```asm
    ldr r0,=TblBase         /* Load MMU translation table base */
    orr r0, r0, #0x5B           /* Outer-cacheable, WB */
    mcr 15, 0, r0, c2, c0, 0        /* TTB0 */
```

Здесь формируется значение регистра с последующей загрузкой непосредственно в регистр. Формат регистра:

|  31:x       | x-1:7    | 6         | 5     | 4:3   | 2     | 1   | 0         |
|-------------|----------|-----------|-------|-------|-------|-----|-----------|
|  `TTB` Addr | Reserved | `IRGN[0]` | `NOS` | `RGN` | `IMP` | `S` | `IRGN[1]` |

`TTB` (Translation Table Base) Address занимает биты `31:x`, где `x = 14 - TTBCR.N`, а `TTBCR.N` согласно документации:

!!! tip "TTBCR.N"

    `N, bits[2:0]` Indicate the width of the base address held in `TTBR0`. In `TTBR0`, the base address field is bits`[31:14-N]`. The value of `N` also determines: 

      * whether `TTBR0` or `TTBR1` is used as the base address for translation table walks. 
      * the size of the translation table pointed to by `TTBR0`. 

    N can take any value from `0` to `7`, that is, from `0b000` to `0b111`. When `N` has its reset value of `0`, the translation table base is compatible with **ARMv5** and **ARMv6**.

Значение по умолчанию `TTBCR.N` равно `0`, что соответствует `x = 14`, т.е. младшие 14 разрядов под биты адреса не используются и при формировании значения адреса предполагаются равными 0 (сама таблица трансляции должна быть выровнена на границу своего размера, т.е. в этом случае размер таблицы 16к и её размещение должно быть выровнено на границу 16к. Это даёт возможность использовать младшие биты для конфигурирования свойств **MMU**.

Остальные поля регистра `TTBR`:

!!! tip "TTBR"

    `NOS, bit[5]`

    Not Outer Shareable bit. Indicates the Outer Shareable attribute for the memory associated with a translation table walk that has the Shareable attribute, indicated by `TTBR0.S == 1`: 

      * `0` Outer Shareable 
      * `1` Inner Shareable. 

    This bit is ignored when `TTBR0.S == 0`. ARMv7 introduces this bit. If an implementation does not distinguish between Inner Shareable and Outer Shareable, this bit is `UNK/SBZP`.

    `RGN, bits[4:3]`

    Region bits. Indicates the Outer cacheability attributes for the memory associated with the translation table walks: 

      * `0b00` Normal memory, Outer Non-cacheable. 
      * `0b01` Normal memory, Outer Write-Back Write-Allocate Cacheable. 
      * `0b10` Normal memory, Outer Write-Through Cacheable. 
      * `0b11` Normal memory, Outer Write-Back no Write-Allocate Cacheable.

    `IMP, bit[2]`

    The effect of this bit is IMPLEMENTATION DEFINED. If the translation table implementation does not include any IMPLEMENTATION DEFINED features this bit is `UNK/SBZP`.

    `S, bit[1]`

    Shareable bit. Indicates the Shareable attribute for the memory associated with the translation table walks: 

      * `0` Non-shareable 
      * `1` Shareable.

    `IRGN, bits[6, 0]`, in an implementation that includes the Multiprocessing Extensions 

    Inner region bits. Indicates the Inner Cacheability attributes for the memory associated with the translation table walks. The possible values of IRGN[1:0] are: 

     * `0b00` Normal memory, Inner Non-cacheable. 
     * `0b01` Normal memory, Inner Write-Back Write-Allocate Cacheable. 
     * `0b10` Normal memory, Inner Write-Through Cacheable. 
     * `0b11` Normal memory, Inner Write-Back no Write-Allocate Cacheable.

Таким образом, значение `0x5B` == 0`b01011011` соответствует значениям полей:

| Name   |  Value   | Description  |
|--------|----------|--------------|
| `IRGN` | `0b11`   | Normal memory, Inner Write-Back no Write-Allocate Cacheable.|
| `S`    | `1`      | Shareable |     
| `IMP`  | `0`      | MMU not include any IMPLEMENTATION DEFINED features |
| `RGN`  | `0b11`   | Normal memory, Outer Write-Back no Write-Allocate Cacheable.|
| `NOS`  | `0`      | Outer Shareable |

Затем идёт настройка `MMU domain`:

```asm
    mvn r0,#0               /* Load MMU domains -- all ones=manager */
    mcr p15,0,r0,c3,c0,0
```
Действие понятно из комментария.

#### Включение MMU и обоих кэшей L1:

```asm
    /* Enable mmu, icahce and dcache */
    ldr r0,=CRValMmuCac
    mcr p15,0,r0,c1,c0,0        /* Enable cache and MMU */
    dsb                         /* dsb  allow the MMU to start up */
    isb                         /* isb  flush prefetch buffer */
```

Здесь в `SCTRL` (это `CP15 c1, c0`) загружается значение `0x1005`, что соответствует установке битов `I` (кэш инструкций), `C` (кэш данных и унифицированный кэш) и `M` (**MMU**). Согласно документации:

!!! tip "**In ARMv7**"


    * **SCTLR.C** enables or disables all data and unified caches for data accesses, across all levels of cache visible to the processor. It is IMPLEMENTATION DEFINED whether it also enables or disables the use of unified caches for instruction accesses.

    * **SCTLR.I** enables or disables all instruction caches, across all levels of cache visible to the processor.

Т.е. биты включения/выключения кэшей работают глобально "сквозь" все уровни кэшей.

#### Иницализация Auxiliary Control Register

!!! tip "**ARMv7-A**"
    
    The ACTLR provides IMPLEMENTATION DEFINED configuration and control options.



```asm
    /* Write to ACTLR */
    mrc p15, 0, r0, c1, c0, 1       /* Read ACTLR*/
    orr r0, r0, #(0x01 << 6)        /* set SMP bit */
    orr r0, r0, #(0x01 )            /* Cache/TLB maintenance broadcast */
    mcr p15, 0, r0, c1, c0, 1       /* Write ACTLR*/
```

Этот код устанавливает в регистре `ACTRL` Cortex-A9 биты `SMP` и `FW`.

|  Name |  Description |
|-------|--------------|
|`SMP`  | Signals if the Cortex-A9 processor is taking part in coherency or not. In uniprocessor configurations, if this bit is set,<br> then Inner Cacheable Shared is treated as Cacheable. The reset value is zero.| 
| `FW`  | Cache and TLB maintenance broadcast:<ul><li>0 Disabled. This is the reset value.</li><li>1 Enabled.</li></ul> `RAZ/WI` if only one Cortex-A9 processor is present.|


#### Таблица трансляции

**MMU** использует инициализированную [таблицу трансляции адресов](mmu.md) (описана в `translation_table.S`)[^1].


### Кэш L2

Кэш `L2` не входит в ядро, и поэтому управляется не через регистры сопроцессора `CP15`, а через регистры, отображаемые на адресное пространство памяти (memory-mapped registers - MMRs).

Выполняемые действия:

```asm
    ldr r0,=L2CCCrtl        /* Load L2CC base address base + control register */
    mov r1, #0              /* force the disable bit */
    str r1, [r0]            /* disable the L2 Caches */
```
Производится загрузка значения 0 в бит 'l2_enable` регистра `reg1_control`, что означает запрещение работы L2 кэша.

Далее:
```asm
   ldr  r0,=L2CCAuxCrtl     /* Load L2CC base address base + Aux control register */
   ldr  r1,[r0]             /* read the register */
   ldr  r2,=L2CCAuxControl  /* set the default bits */
   orr  r1,r1,r2
   str  r1, [r0]            /* store the Aux Control Register */
```
Считывается значение регистра `reg1_aux_control`, оно соответствует значению по умолчанию: `0x02060000`. В дополнение к этому устанавливаются ещё биты `0x72360000`, что даёт в результате то же самое `0x72360000`:

| Name | Bit | Description |
|------|-----:|-------------|
| way_size          | 19:17 | Way-size, `0x3` = `64KB` |
| event_mon_bus_en  | 20 | Event monitor bus enable, `1` = Enabled |
| parity_en         | 21 | Parity enable, `1` = Enabled |
| data_prefetch_en  | 28 | Data prefetch enable, `1` = Enabled |
| instr_prefetch_en | 29 | Instruction prefetch enable, `1` = Enabled |
| early_bresp_en    | 30 | Early BRESP enable, `1` = Enabled |

Затем в регистр `reg1_tag_ram_control` загружается значение `0x00000111`:
```asm
   ldr  r0,=L2CCTAGLatReg       /* Load L2CC base address base + TAG Latency address */
   ldr  r1,=L2CCTAGLatency      /* set the latencies for the TAG*/
   str  r1, [r0]                /* store the TAG Latency register Register */
```
что устанавливает значения полей регистра:
```
ram_setup_lat     = 0x1
ram_rd_access_lat = 0x1
ram_wr_access_lat = 0x1
```
Значение `0x1` для любого из этих полей соответствует `2 cycles latency`.

Далее аналогичная загрузка производится в регистр `reg1_data_ram_control`:
```asm
   ldr  r0,=L2CCDataLatReg      /* Load L2CC base address base + Data Latency address */
   ldr  r1,=L2CCDataLatency     /* set the latencies for the Data*/
   str  r1, [r0]                /* store the Data Latency register Register */
```
только значение загружается `0x00000121`, что соответствует:
```
ram_setup_lat     = 0x1 (2 cycles latency)
ram_rd_access_lat = 0x2 (3 cycles latency)
ram_wr_access_lat = 0x1 (2 cycles latency)
```

После этого загружается регистр `reg7_inv_way` значением `0x0000ffff`, что инвалидирут все каналы (ways) кэша:
```asm
   ldr  r0,=L2CCWay         /* Load L2CC base address base + way register*/
   ldr  r2, =0xFFFF
   str  r2, [r0]            /* force invalidate */
```
Тут не совсем понятно: в этом регистре значимые биты только `7:0`, каждый бит отвечает за свой канал, а поскольку L2 кэш является 8-канальным, то и битов, соответствующих каналам, тоже 8. Т.е. загружаемое значение должно быть `0x000000ff`. Старшие биты `31:8` в документации обозначены как `reserved`. Похоже на баг.

После инвалидации производится цикл ожидания, который называется почему-то синхронизацией. По факту цикле читается значение регистра `reg7_cache_sync`, в котором только один значимый бит `c`:

!!! tip "**Cache Sync**"
    
    **c**: Cache Sync: Drain the STB. Operation complete when all buffers, `LRB`, `LFB`, `STB` and `EB`, are empty.

Код опроса регистра:
```asm
   ldr  r0,=L2CCSync            /* need to poll 0x730, PSS_L2CC_CACHE_SYNC_OFFSET */
                                /* Load L2CC base address base + sync register*/
   /* poll for completion */
Sync:
   ldr  r1, [r0]
   cmp  r1, #0
   bne  Sync
```

Далее производится сброс битов (флагов) прерываний:
```asm
    ldr r0,=L2CCIntRaw          /* clear pending interrupts */
    ldr r1,[r0]
    ldr r0,=L2CCIntClear
    str r1,[r0]
```
Тут читается значение статусного регистра прерываний и затем оно записывается в регистр сброса флагов прерываний&nbsp;– сброс флага производится путём записи `1`, т.е. если какой-либо флаг был установлен, он будет сброшен.

Завершающая цепочка действий:

```asm
    ldr r0,=SLCRUnlockReg       /* Load SLCR base address base + unlock register */
    ldr r1,=SLCRUnlockKey       /* set unlock key */
    str r1, [r0]                /* Unlock SLCR */

    ldr r0,=SLCRL2cRamReg       /* Load SLCR base address base + l2c Ram Control register */
    ldr r1,=SLCRL2cRamConfig    /* set the configuration value */
    str r1, [r0]                /* store the L2c Ram Control Register */

    ldr r0,=SLCRlockReg         /* Load SLCR base address base + lock register */
    ldr r1,=SLCRlockKey         /* set lock key */
    str r1, [r0]                /* lock SLCR */

    ldr r0,=L2CCCrtl            /* Load L2CC base address base + control register */
    ldr r1,[r0]                 /* read the register */
    mov r2, #L2CCControl        /* set the enable bit */
    orr r1,r1,r2
    str r1, [r0]                /* enable the L2 Caches */
```
Здесь осуществляется разблокировка доступа по записи в регистры группы `SCLR`, запись "магического" значения `0x0020202` (такое значение требуется по документации) в регистр по адресу `0xf000a1c` со странным названием `reserved` и последующая блокировка. Последнее действие - разрешение работы L2 кэша.

### Завершение низкоуровневой инициализации

#### Разрешение работы сопроцессоров CP10 и CP11

```asm
    mov r0, r0
    mrc p15, 0, r1, c1, c0, 2       /* read cp access control register (CACR) into r1 */
    orr r1, r1, #(0xf << 20)        /* enable full access for p10 & p11 */
    mcr p15, 0, r1, c1, c0, 2       /* write back into CACR */
```

#### Разрешение работы VFP

```asm
    /* enable vfp */
    fmrx    r1, FPEXC           /* read the exception register */
    orr r1,r1, #FPEXC_EN        /* set VFP enable bit, leave the others in orig state */
    fmxr    FPEXC, r1           /* write back the exception register */
```

#### Включение предсказателя ветвлений

```asm
    mrc p15,0,r0,c1,c0,0        /* flow prediction enable */
    orr r0, r0, #(0x01 << 11)   /* #0x8000 */
    mcr p15,0,r0,c1,c0,0
```

#### Коррекция настроек кэша L2

```asm
    mrc p15,0,r0,c1,c0,1        /* read Auxiliary Control Register */
    orr r0, r0, #(0x1 << 2)     /* enable Dside prefetch */
    orr r0, r0, #(0x1 << 1)     /* enable L2 Prefetch hint */
    mcr p15,0,r0,c1,c0,1        /* write Auxiliary Control Register */
```

#### Разрешение исключений типа Abort

```asm
    mrs r0, cpsr                /* get the current PSR */
    bic r0, r0, #0x100          /* enable asynchronous abort exception */
    msr cpsr_xsf, r0
```


#### Приведение некоторых регистров сопроцессоров в детерминированное состояние

```
   // Clear cp15 regs with unknown reset values
    mov r0, #0x0
    mcr p15, 0, r0, c5, c0, 0         //  DFSR
    mcr p15, 0, r0, c5, c0, 1         //  IFSR
    mcr p15, 0, r0, c6, c0, 0         //  DFAR
    mcr p15, 0, r0, c6, c0, 2         //  IFAR
    mcr p15, 0, r0, c9, c13, 2        //  PMXEVCNTR
    mcr p15, 0, r0, c13, c0, 2        //  TPIDRURW
    mcr p15, 0, r0, c13, c0, 3        //  TPIDRURO

    // Reset and start Cycle Counter
    mov r2, #0x80000000               //  clear overflow */
    mcr p15, 0, r2, c9, c12, 3        //
    mov r2, #0xd                      //  D, C, E */
    mcr p15, 0, r2, c9, c12, 0        //
    mov r2, #0x80000000               //  enable cycle counter */
    mcr p15, 0, r2, c9, c12, 1        //

```

Действия в первой части понятны из комментариев. Во второй части:

| Instruction | Register | Description |
|-------------|----------|-------------|
| `mcr   p15, 0, r2, c9, c12, 3` | `PMOVSR` | Overflow Flag Status Register. Запись `0x80000000`<br>сбрасывает бит `C`, который является флагом<br>переполнения счётчика циклов (Cycle Counter) | 
| `mcr   p15, 0, r2, c9, c12, 0` | `PMCR` | Performance Monitor Control Register. Запись `0xd` означает:<ul><li>`E`: All counters enabled</li><li>`C`: Cycle counter reset</li><li>`D`: Cycle counter clock divider, `1` = `PMCCNTR` counts once<br>every 64 cycles, `0` = `PMCCNTR` counts every clock cycle</li></ul>|
| `mcr   p15, 0, r2, c9, c12, 1` | `PMCNTENSET` | Performance Monitors Count Enable Set register.<br>Запись значения `0x80000000` устанавливает бит `C`,<br>который включает `PMCCONTR` cycle counter.|


## Старт программы

После этого управление передаётся подпрограмме `_start` (исходный файл `startup.c`), который выполняет функции стандартного стартапа любой `C/C++` программы:

  * статическую инициализацию (инициализация глобальных переменных);
  * динамическую инициализацию (вызов конструкторов глобальных объектов).

Но перед этим осуществляется инициализация периферийных устройств путём вызова сгенерированной пакетом **Vivado** функцией `ps7_init()`.

Исходный код этой функции простой и компактный:

```cpp
void _start()
{
    if( __low_level_init() )
    {
        ps7_init();                                                                   //
        memset(__bss_start, 0, __bss_end - __bss_start);                              // zero-fill uninitialized variables
        memset(__ddr_code_start, 0, __ddr_code_end - __ddr_code_start + 32);          // copy initialized variables
        memcpy(__ddr_code_start, __ddr_src_start, __ddr_code_end - __ddr_code_start); // copy initialized variables
        Xil_DCacheFlush();
        __libc_init_array();                                                          // low-level init & ctor loop
    }
    main();
}

```

### Инициализация периферийных устройств PS

#### Общие сведения

Инициализация периферийных устройств организована достаточно просто: инструментальные средства **Vivado** генерируют файлы, код которых отвечает за инициализацию всей периферии:

|  File          |  Description       |
|----------------|--------------------|
| `ps7_init.h`   | Определения макросов и прототипов функций |
| `ps7_init.c`   | Код и данные для инициализации |
| `ps7_init.tcl` | Код для IDE для эмуляции инициализации загрузчиком, <br>необходим для отладки пользовательского приложения |

#### Процесс инициализации

##### Краткое описание

Процесс инициализации осуществляется путём вызова функции `ps7_init()`. Функция состоит из двух логических частей:

1. Подготовка данных.
1. Инициализация.

Подготовка данных сводится к выбору в соответствии с ревизией используемой целевой микросхемы SoC набора данных, это осуществляется путём присвоения соответствующих значений указателям на структуры данных.

Собственно инициализация выполняется путём загрузки данных в регистры периферийных устройств. Процесс загрузки организован как выполнение простых инструкций неким виртуальным процессором. Сами инструкции "записаны" в структурах данных. Такой подход обеспечивает относительную простоту реализации и универсальность, хотя и даёт заметные накладные расходы как по размеру данных, так и по времени выполнения.

##### Структуры данных

В файле `ps7_init.h` определены следующие макросы:

```C
#define OPCODE_EXIT       0U
#define OPCODE_CLEAR      1U
#define OPCODE_WRITE      2U
#define OPCODE_MASKWRITE  3U
#define OPCODE_MASKPOLL   4U
#define OPCODE_MASKDELAY  5U

#define EMIT_EXIT()                   ( (OPCODE_EXIT      << 4 ) | 0 )
#define EMIT_CLEAR(addr)              ( (OPCODE_CLEAR     << 4 ) | 1 ) , addr
#define EMIT_WRITE(addr,val)          ( (OPCODE_WRITE     << 4 ) | 2 ) , addr, val
#define EMIT_MASKWRITE(addr,mask,val) ( (OPCODE_MASKWRITE << 4 ) | 3 ) , addr, mask, val
#define EMIT_MASKPOLL(addr,mask)      ( (OPCODE_MASKPOLL  << 4 ) | 2 ) , addr, mask
#define EMIT_MASKDELAY(addr,mask)     ( (OPCODE_MASKDELAY << 4 ) | 2 ) , addr, mask
```

Они используются для формирования значений элементов структур данных. Собственно, структуры данных представляют собой массивы, состоящие из цепочек слов, образующих "инструкции" некоего виртуального процессора. Длина цепочек-инструкций варьируется от 1 до 4. Первое слово всегда комбинация опкода (старший нибл младшего байта) и количества аргументов (младший нибл младшего байта). Из определения макросов видно, что инструкция `EXIT` имеет длину в одно слово, `CLEAR` - 2 слова, `MASKWRITE` - 4 слова, остальные - по 3 слова.

Все массивы инструкций организованы одинаково - см. пример (фрагмент (начало и окончание) массива для инициализации устройства ФАПЧ):
```C
unsigned long ps7_pll_init_data_3_0[] = {
    // START: top
    // .. START: SLCR SETTINGS
    // .. UNLOCK_KEY = 0XDF0D
    // .. ==> 0XF8000008[15:0] = 0x0000DF0DU
    // ..     ==> MASK : 0x0000FFFFU    VAL : 0x0000DF0DU
    // .. 
    EMIT_MASKWRITE(0XF8000008, 0x0000FFFFU ,0x0000DF0DU),
    // .. FINISH: SLCR SETTINGS
    // .. START: PLL SLCR REGISTERS
    // .. .. START: ARM PLL INIT
    // .. .. PLL_RES = 0x4
    // .. .. ==> 0XF8000110[7:4] = 0x00000004U
    // .. ..     ==> MASK : 0x000000F0U    VAL : 0x00000040U
    // .. .. PLL_CP = 0x2
    // .. .. ==> 0XF8000110[11:8] = 0x00000002U
    // .. ..     ==> MASK : 0x00000F00U    VAL : 0x00000200U
    // .. .. LOCK_CNT = 0xfa
    // .. .. ==> 0XF8000110[21:12] = 0x000000FAU
    // .. ..     ==> MASK : 0x003FF000U    VAL : 0x000FA000U
    // .. .. 
    EMIT_MASKWRITE(0XF8000110, 0x003FFFF0U ,0x000FA240U),
    ...
    ...
    ...
    ...
    //
    EMIT_EXIT(),
};
```
В этом примере макрос `EMIT_MASKWRITE(0XF8000008, 0x0000FFFFU ,0x0000DF0DU)` генерирует последовательность слов:

1. `0x00000033`: опкод и количество аргументов
1. `0xf8000008`: адрес регистра
1. `0x0000ffff`: маска, накладываемая на записываемое в регистр слово данных
1. `0x0000df0d`: слово данных

При применении этой "инструкции" про указанном адресу будет записано число `0xdf0d` в младшие 16 бит регистра, старшие 16 бит останутся без изменений, это обеспечивается применением маски - обновляются только те биты, в позиции которых в значении маски стоят 1. В данном конкретном случае осуществляется запись в регистр `SCLR_UNLOCK` специального кода, который разблокирует доступ по записи к группе регистров `sclr`.

Остальные инструкции генерируются аналогично. Для удобства в комментариях к инструкциям поясняется, какие биты, флаги, поля регистров модифицируются.

Размер файла `ps7_init.c` достаточно большой (более полумегабайта), но страшного в этом ничего нет - львиную долю этого объёма занимаются массивы инструкций и комментарии к ним, а таких массивов там полтора десятка: три ревизии, для каждой инициализация  MIO, PLL, Clock, DDR, Peripherals.

##### Загрузка

Весь процесс инициализации сводится к выполнению инструкций из массивов:
```C
int
ps7_init() 
{
  // Get the PS_VERSION on run time
  unsigned long si_ver = ps7GetSiliconVersion ();
  int ret;

  // инициализация указателей на массивы инструкций
  // в соответствии с ревизией кристалла
  // ...  
  // ...  
  // ...  

  // MIO init
  ret = ps7_config (ps7_mio_init_data);  
  if (ret != PS7_INIT_SUCCESS) return ret;

  // PLL init
  ret = ps7_config (ps7_pll_init_data); 
  if (ret != PS7_INIT_SUCCESS) return ret;

  // Clock init
  ret = ps7_config (ps7_clock_init_data);
  if (ret != PS7_INIT_SUCCESS) return ret;

  // DDR init
  ret = ps7_config (ps7_ddr_init_data);
  if (ret != PS7_INIT_SUCCESS) return ret;

  // Peripherals init
  ret = ps7_config (ps7_peripherals_init_data);
  if (ret != PS7_INIT_SUCCESS) return ret;
  //xil_printf ("\n PCW Silicon Version : %d.0", pcw_ver);
  return PS7_INIT_SUCCESS;
}
```
Из кода функции видно, что сначала инициализируются регистры `MIO`, затем `PLL`, дерево тактовых частот, `DDR`, периферийных устройств. Сам процесс загрузки данных в регистры во всех случаях один и тот же и выполняется с помощью функции `ps7_config()`. Фрагмент этой функции, отвечающий за собственно выполнение инструкций:

```C
     ...   
     ...   
     ...   
     switch ( opcode ) {
            
        case OPCODE_EXIT:
            finish = PS7_INIT_SUCCESS;
            break;
            
        case OPCODE_CLEAR:
            addr = (unsigned long*) args[0];
            *addr = 0;
            break;

        case OPCODE_WRITE:
            addr = (unsigned long*) args[0];
            val = args[1];
            *addr = val;
            break;

        case OPCODE_MASKWRITE:
            addr = (unsigned long*) args[0];
            mask = args[1];
            val = args[2];
            *addr = ( val & mask ) | ( *addr & ~mask);
            break;

        case OPCODE_MASKPOLL:
            addr = (unsigned long*) args[0];
            mask = args[1];
            i = 0;
            while (!(*addr & mask)) {
                if (i == PS7_MASK_POLL_TIME) {
                    finish = PS7_INIT_TIMEOUT;
                    break;
                }
                i++;
            }
            break;
        case OPCODE_MASKDELAY:
            addr = (unsigned long*) args[0];
            mask = args[1];
            int delay = get_number_of_cycles_for_delay(mask);
            perf_reset_and_start_timer(); 
            while ((*addr < delay)) {
            }
            break;
        default:
            finish = PS7_INIT_CORRUPT;
            break;
        }
     ...   
     ...   
     ...   
```
Инструкция с опкодом `OPCODE_EXIT` завершает циклический процесс выполнения инструкций. Такая инструкция помещается в конец массива.

Инструкция с опкодом `OPCODE_CLEAR` загружает в регистр по адресу значение `0`.

Инструкция с опкодом `OPCODE_WRITE` загружает в регистр по адресу значение аргумента, переданного вслед за опкодом.

Инструкция с опкодом `OPCODE_MASK_WRITE` загружает в регистр по адресу значение аргумента в соответствии с маской: загружаются только те биты значения, в позиции которых в значении маски стоят `1`.

Инструкция с опкодом `OPCODE_MASKPOLL` выполняет следующие действия: в цикле читает значение регистра по адресу, накладывает на это значение маску и проверяет на ноль. Как только значение регистра с маской дадут ноль, опрос прекращается. Количество циклов в опросе задаётся макросом `PS7_MASK_POLL_TIME` и составляет `100000000`. Инструкция служит для опроса значений регистров с целью определить, установился ли тот или иной бит, флаг и поле.

Инструкция с опкодом `OPCODE_MASKDELAY` просто формирует задержку, длительность которой определяется значением аргумента инструкции. Задержка задаётся в миллисекундах. Задержка формируется путём сравнения значения счётного регистра глобального таймера с вычисленным значением, соответствующим указанной задержке в миллисекундах. Таймер останавливается, его счётный регистр обнуляется, затем таймер запускается и производится опрос счётного регистра таймера на предмет превышения порогового значения, после чего выполнение этой инструкции завершается. Инструкция используется для формирования задержек реального времени.


Обычное для этого места действие&nbsp;– инициализация стека не производится, т.к. указатели стеков (для всех режимов процессора) проинициализированы ранее.

После этого управление передаётся функции `main`.

## Функция main

Функция `main` выполняет подготовительные действия, необходимые для запуска основной прикладной программы, по окончании которых передаёт управление этой программе.

Код функции `main` (и функции `load_img`, которая собственно осуществляет загрузку прикладной программы из флешь памяти в **OCM**) размещается в **DDR** памяти. Это сделано из-за необходимости перемещения сегментов **OCM** (три сегмента **OCM** перемещаются с адреса `0x00000000` в адрес `0xfffc0000`, образуя с четвёртым сегментом, расположенным по адресу `0xffff0000` непрерывную область памяти размером 256 кбайт)&nbsp;– работающие код и данные не могут размещаться в перемещаемых сегментах, поэтому они помещены в **DDR** память.

Первое действие: перемещение таблицы трансляции **MMU** в старшие адреса **OCM**&nbsp;– это необходимо сделать, т.к. в дальнейшем все сегменты **OCM** будут перемещены в старшие адреса памяти, и текущее положение таблицы трансляции окажется не валидным.

Затем выполняется настройка системы прерываний:

  * инициализация массива указателей на обработчики прерываний;
  * установка приоритетов прерываний;
  * назначение целевого процессора для прерываний от периферийных устройств;
  * разрешение прерываний.  

В процессе работы функции производится печать сообщений в терминал. По окончании передачи всех символов сообщений (программа специально ждёт этого&nbsp;– это важно) производится переключение сегментов **OCM** с младших адресов в старшие. Для того, чтобы переключение не привело к ошибкам работы программы, необходимо, чтобы в перемещаемых сегментах не было работающего кода. Поэтому функция `main` (и другие функции, которые она использует после переключения) выполняется из DDR памяти. В режиме отладки этот код сразу загружается в DDR память, при автономной работе это делает функция стартапа перед передачей управления в функцию `main`.

После этого SoC готов к загрузке прикладной основной программы, которая осуществляется путём вызова функции `load_img`.

### Загрузка прикладной программы

Образ прикладной программы хранится во внешней флешь памяти...

...

При разработке и отладке прикладной программы рекомендуется использовать [эмуляцию работы загрузчика](debug.md/#debug-load).



  [^1]: Следует отметить, что таблица трансляции **MMU** не загружается из образа прикладной программы, а помещается по указанному адресу путём таблицы трансляции **MMU** самого загрузчика&nbsp;– это возможно благодаря тому, что содержимое таблицы трансляции **MMU** загрузчика и прикладной программы идентичны.