## Таблица трансляции адресов

### Общие сведения

`MMU` в `ARMv7` поддерживает одноуровневое и двухуровневое преобразование адресов, соответственно для каждого уровня существует своя отдельная таблица. Таблица первого уровня имеет размер `16kB` (и должна быть размещена в памяти по адресу, выровненному на границу `16kB`). Если преобразование одноуровневое, то таблица второго уровня не нужна. Таблица первого уровня нужна всегда, когда `MMU` включено. 

### Таблица трансляции первого уровня

Таблица трансляции адресов инициализирована значениями, соответствующими кодированию 1М секций. Назначение таблицы в данном случае - задание атрибутов различных регионов адресного пространства.

Ниже приведено распределение значений атрибутов по адресам секции. В столбце `Address` указаны смещения, т.е реальный физический адрес элемента таблицы получается получается путём прибавления смещения к базовому адресу таблицы. `SA` - Section Address, адрес секции, у каждого последующего элемента таблицы `SA` равен адресу из предыдущего элемента + `0x100000` (1MB - т.е. на размер секции).

|  Entry Address Offset  | Sections | Value    | Destination Address | Device   | Description |
|------------------------|------|--------------|-------------------------|-------|-------------|
|`0x00000000-0x00000fff` | 1024 | `SA + 0x15de6` | `0x00000000-0x3fffffff` | OCRAM/DDR   | S=b1 TEX=b101 AP=b11, <br>Domain=b1111, C=b0, B=b1 |
|`0x00001000-0x00001fff` | 1024 | `SA + 0x00с02` | `0x40000000-0x7fffffff` | FPGA Slave0 | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b0 |
|`0x00002000-0x00002fff` | 1024 | `SA + 0x00с02` | `0x80000000-0xbfffffff` | FPGA Slave1 | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b0 |
|`0x00003000-0x000037ff` | 512  | `SA + 0x00000` | `0xc0000000-0xdfffffff` | Reserved    | S=b0 TEX=b000 AP=b00, <br>Domain=b0, C=b0, B=b0 |
|`0x00003800-0x0000380b` |  3   | `SA + 0x00c06` | `0xe0000000-0xe02fffff` | MMRs    | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b1 |
|`0x0000380c-0x0000383f` | 13   | `SA + 0x00000` | `0xe0300000-0xe0ffffff` | Reserved    | S=b0 TEX=b000 AP=b00, <br>Domain=b0, C=b0, B=b0 |
|`0x00003840-0x0000387f` | 16   | `SA + 0x00c06` | `0xe1000000-0xe1ffffff` | NAND    | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b1 |
|`0x00003880-0x000038ff` | 32   | `SA + 0x00c06` | `0xe2000000-0xe3ffffff` | NOR    | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b1 |
|`0x00003900-0x0000397f` | 32   | `SA + 0x00c06` | `0xe4000000-0xe5ffffff` | SRAM    | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b1 |
|`0x00003980-0x00003dff` | 288  | `SA + 0x00000` | `0xe6000000-0xf7ffffff` | Reserved    | S=b0 TEX=b000 AP=b00, <br>Domain=b0, C=b0, B=b0 |
|`0x00003e00-0x00003e3f` | 16   | `SA + 0x00c06` | `0xf8000000-0xf8ffffff` | APB Peripherals    | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b0, B=b1 |
|`0x00003e40-0x00003eff` | 48  | `SA + 0x00000` | `0xf9000000-0xfbffffff`  | Reserved    | S=b0 TEX=b000 AP=b00, <br>Domain=b0, C=b0, B=b0 |
|`0x00003f00-0x00003f7f` | 32  | `SA + 0x00c0a` | `0xfc000000 - 0xfdffffff` | Linear QSPI | S=b0 TEX=b000 AP=b11, <br>Domain=b0, C=b1, B=b0 |
|`0x00003f80-0x00003ffb` | 31  | `SA + 0x00000` | `0xfe000000-0xffefffff` | Reserved    | S=b0 TEX=b000 AP=b00, <br>Domain=b0, C=b0, B=b0 |
|`0x00003ffc-0x00003fff` | 1  | `SA + 0x04c0e` | `0xfff00000-0xffffffff` | 256K OCM | S=b0 TEX=b100 AP=b11, <br>Domain=b0, C=b1, B=b1 |

### Расшифровка атрибутов

| ID       |  Name     | Description |
|----------|-----------|-------------|
| `S`        | Shareable | Регион доступен не только ядру, но и другим агентам (например, другому ядру). <ul><li>`1`: Shareable</li><li>`0`: Non-shareable</li></ul>  |
| `C`        | Cacheable | Регион кэшируем. В сочетании со значением атрибута `TEX` относится к внутреннему или <br>внешнему кэшу (Inner-Cacheable, Outer-Cacheable).<ul><li>`1`: Cacheable</li><li>`0`: Non-сacheable</li></ul>  |
| `B`        | Bufferable | Регион буферируем. Имеет смысл при запросах ядра на запись.<ul><li>`1`: Bufferable</li><li>`0`: Non-bufferable</li></ul>  |
| `TEX`      | Type extension | Атрибут (вместе с атрибутами `C`, `B` и частично `S`) задаёт модель памяти (Memory Ordering). <br>Полный перечень вариантов описан в документации, тут приведено описание только <br>используемых случаев.<ul><li>`0b000`:<ul><li>`C`= 0; `B` = 0: Strong-ordered.</li><li>`C`= 0; `B` = 1: Shareable Device.</li><li>`C`= 1; `B` = 0: Normal, Outer and Inner Write-Through, no Write-Allocate.</li><li>`C`= 1; `B` = 1: Normal, Outer and Inner Write-Back, no Write-Allocate.</li></ul></li><li>`0b1xx`: `0bxx` определяет Outer cacheable attribute, `C` и `B` определяют Inner cacheable <br>attribute. Кодировка `0bxx`:<ul><li>`0b00`: Non-cacheable.</li><li>`0b01`: Write-Back, Write-Allocate.</li><li>`0b10`: Write-Through, no Write-Allocate.</li><li>`0b11`: Write-Back, no Write-Allocate.</li></ul></li></ul> |
| `AP` | Access permissions | Опции доступа к региону памяти. Если `APX == 0`, а это именно рассматриваемый случай, <br>то кодировка атрибута `AP` для привилегированного режима (`PL1`) следующая:<ul><li>`0b00`: No access.</li><li>`0b01, 0b10, 0b11`: Read/Write.</li></ul>
| `Domain` | Domain field | Дескриптор домена памяти. Всего существует 16 доменов, каждый домен характеризуется <br>правами доступа, определяемыми в `DACR`&nbsp;– Domain Access Control Register. Этот регистр <br>является 32-разрядным, и поделён на 2-битные поля, полей 16 штук - по числу доменов. <br>Младшие 2 бита относятся к домену 0, следующие 2 - к домену 1 и т.д. Это двухбитное <br>поле кодирует права доступа к домену следующим образом:<ul><li>`0b00`: No access. Any access to the domain generates a Domain fault.</li><li>`0b01`: Client. Accesses are checked against the permission bits in the translation tables.</li><li>`0b10`: Reserved, effect is UNPREDICTABLE.</li><li>`0b11`: Manager. Accesses are not checked against the permission bits in the translation tables.</li></ul> |
| nG  | Non-Global | Имеет смысл в многопроцессной ОС. Если если значение равно `0`, то регион доступен всем <br>процессам, если значение равно `1`, то регион доступен только одному процессу - в этом <br>случае дополнительно используется `Address Space ID` (`ASID`) |
| xN  | Execute Never | Будучи установленным, предотвращает спекулятивное чтение инструкций из региона памяти. <br>Типовое применение&nbsp;– обозначить `device memory regions` для защиты от случайной попытки выполнения программы из этих регионов. |

### Резюме

Все домены в данной программе установлены в режим `Manager`, поэтому доступ к ним разрешён безотносительно к значениям атрибутов `AP`, указанных в дескрипторах секций.

В вышеописанной таблице все регионы (секции) памяти описаны с использованием следующего набора значений (приведена только часть, описывающая свойства секции, без адреса:

|  Value    | Attributes | Description | Device |
|-----------|------------|-------------|-------------|
| `0x15de6` | `S=b1`<br>`TEX=b101`<br>`AP=b11`<br>`Domain=b1111`<br>`C=b1`<br>`B=b1` | Память является:<ul><li>`Normal`</li><li>`Inner`: `Write-Back`, no `Write-Allocate`</li><li>`Outer`: `Write-Back`, `Write-Allocate`</li></ul>  | OCR/DDR |
| `0x00000` | `S=b0`<br>`TEX=b000`<br>`AP=b00`<br>`Domain=b0`<br>`C=b0`<br>`B=b0` | Память является `Strongly-ordered` | Reserved |
| `0x00c02` | `S=b0`<br>`TEX=b000`<br>`AP=b11`<br>`Domain=b0`<br>`C=b0`<br>`B=b0` | Память является `Strongly-ordered`  | FPGA:<ul><li>`Slave 0`</li><li>`Slave 1`</li></ul> |
| `0x00c06` | `S=b0`<br>`TEX=b000`<br>`AP=b11`<br>`Domain=b0`<br>`C=b0`<br>`B=b1` | Память является `Device`  | Memory mapped devices:<ul><li>`UART`</li><li>`USB`</li><li>`IIC`</li><li>`SPI`</li><li>`CAN`</li><li>`GEM`</li><li>`GPIO`</li><li>`QSPI`</li><li>`SD`</li><li>`NAND`</li><li>`NOR`</li><li>`APB Peripherals`</li></ul> |
| `0x00c0a` | `S=b0`<br>`TEX=b000`<br>`AP=b11`<br>`Domain=b0`<br>`C=b1`<br>`B=b0` | Память является:<ul><li>`Normal`</li><li>`Inner`: `Write-Through`, no `Write-Allocate`</li><li>`Outer`: `Write-Through`, no `Write-Allocate`</li></ul> | Linear `QSPI` - `XIP` |
| `0x04c0e` | `S=b0`<br>`TEX=b100`<br>`AP=b11`<br>`Domain=b0`<br>`C=b1`<br>`B=b1` | Память является:<ul><li>`Normal`</li><li>`Inner`: `Write-Back`, no `Write-Allocate`</li><li>`Outer`: `Non-cacheable`</li></ul>| `OCM` когда отображена на старшие адреса |
