Процесс загрузки ядра. Часть 4.
================================================================================

Переход в 64-битный режим
--------------------------------------------------------------------------------

Это четвёртая часть `Процесса загрузки ядра`, в которой вы увидите первые шаги в [защищённом режиме](http://en.wikipedia.org/wiki/Protected_mode), такие как проверка поддержки процессором [long mode](http://en.wikipedia.org/wiki/Long_mode) и [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions), [страничная организация памяти](http://en.wikipedia.org/wiki/Paging), инициализация таблиц страниц и в конце мы обсудим переход в [long mode](https://en.wikipedia.org/wiki/Long_mode).

**ЗАМЕЧАНИЕ: данная часть содержит много ассемблерного кода, так что если вы не знакомы с ним, вы можете прочитать соответствующую литературу**

В предыдущей [части](linux-bootstrap-3.md) мы остановились на переходе к 32-битной точке входа в [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S):

```assembly
jmpl	*%eax
```

Вы помните, что регистр `eax` содержит адрес 32-битной точки входа. Мы можем прочитать об этом в [протоколе загрузки ядра Linux x86](https://www.kernel.org/doc/Documentation/x86/boot.txt):

```
При использовании bzImage ядро в защищённом режиме перемещается на 0x100000
```

Давайте удостоверимся в том, что это правда, посмотрев на значения регистров в 32-битной точке входа:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

Мы видим, что регистр `cs` содержит `0x10` (как вы помните из предыдущей части, это второй индекс в глобальной таблице дескрипторов), регистр `eip` содержит `0x100000`, и базовый адрес всех сегментов, в том числе сегмента кода, равен нулю. Таким образом, мы можем получить физический адрес - это будет `0:0x100000` или просто `0x100000`, как указано в протоколе загрузки. Давайте начнём с 32-битной точки входа.

32-битная точка входа
--------------------------------------------------------------------------------

Мы можем найти определение 32-битной точки входа в [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S):

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

Прежде всего, почему директория `compressed`? На самом деле, `bzimage` является сжатым `vmlinux + заголовок + код настройки ядра`. Мы видели код настройки ядра во всех предыдущих частях. Таким образом, главная цель `head_64.S` - подготовка перехода в lоng mode, переход в него и декомпрессия ядра. В этой части мы увидим все шаги, вплоть до декомпрессии ядра.

В директории `arch/x86/boot/compressed` содержится два файла:

* [head_32.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)

но мы будем рассматривать только `head_64.S`, потому что, как вы помните, эта книга только о `x86_64`; `head_32.S` в нашем случае не используется. Давайте посмотрим на [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile). Здесь мы можем увидить следующую цель сборки:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

Обратите внимание на `$(obj)/head_$(BITS).o`. Это означает, что выбор файла (head_32.o или head_64.o) для линковки будет зависеть от значения `$(BITS)`. `$(BITS)` определён в [arch/x86/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/Makefile), основанном на .config файле:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

Перезагрузка сегментов, если это необходимо
--------------------------------------------------------------------------------

Как было отмечено выше, мы начинаем с ассемблерного файла [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S). Во-первых, мы видим определение специального атрибута секции перед определением `startup_32`:

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

`__HEAD` является макросом, определённым в [include/linux/init.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/init.h) и представляет собой следующую секцию:

```C
#define __HEAD		.section	".head.text","ax"
```

с именем `.head.text` и флагами `ax`. В нашем случае эти флаги означают, что секция является [исполняемой](https://en.wikipedia.org/wiki/Executable) или, другими словами, содержит код. Мы можем найти определение этой секции в скрипте компоновщика [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S):

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
     }
     ...
     ...
     ...
}
```

Если вы не знакомы с синтаксисом скриптового языка компоновщика `GNU LD`, вы можете найти более подробную информацию в [документации](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts). Вкратце, символ `.` является специальной переменной компоновщика - счётчиком местоположения. Значение, присвоенное ему - это смещение по отношению к смещению сегмента. В нашем случае мы устанавливаем счётчик местоположения в ноль. Это означает, что наш код слинкован для запуска в памяти со смещения `0`. Кроме того, мы можем найти эту информацию в комментарии:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```

Хорошо, теперь мы знаем, где мы находимся, и сейчас самое время заглянуть внутрь функции `startup_32`.

В начале `startup_32` мы видим инструкцию `cld`, которая очищает бит `DF` в [регистре флагов](https://en.wikipedia.org/wiki/FLAGS_register). Когда флаг направления очищен, все строковые операции, такие как [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) и др. будут инкрементировать индексные регистры `esi` или `edi`. Нам нужно очистить флаг направления, потому что позже мы будем использовать строковые операции для очистки пространства для таблиц страниц и т.д.

После того как бит `DF` очищен, следующим шагом является проверка флага `KEEP_SEGMENTS` из поля `loadflags` заголовка настройки ядра. Если вы помните, мы уже видели `loadflags` в самой первой [части](linux-bootstrap-1.md) книги. Там мы проверяли флаг `CAN_USE_HEAP` чтобы узнать, можем ли мы использовать кучу. Теперь нам нужно проверить флаг `KEEP_SEGMENTS`. Данный флаг описан в [протоколе загрузки](https://www.kernel.org/doc/Documentation/x86/boot.txt):

```
Бит 6 (запись): KEEP_SEGMENTS
  Протокол: 2.07+
  - Если 0, перезагрузить регистры сегмента в 32-битной точке входа.
  - Если 1, не перезагружать регистры сегмента в 32-битной точке входа.
    Предполагается, что %cs %ds %ss %es установлены в плоские сегменты
    с базовым адресом 0 (или эквивалент для их среды).
```

Таким образом, если бит `KEEP_SEGMENTS` в `loadflags` не установлен, то сегментные регистры `ds`, `ss` и `es` должны быть установлены в индекс сегмента данных с базовым адресом `0`. Что мы и делаем:

```C
	testb $KEEP_SEGMENTS, BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

Вы помните, что `__BOOT_DS` равен `0x18` (индекс сегмента данных в [глобальной таблице дескрипторов](https://en.wikipedia.org/wiki/Global_Descriptor_Table)). Если `KEEP_SEGMENTS` установлен, мы переходим на ближайшую метку `1f`, иначе обновляем сегментные регистры значением `__BOOT_DS`. Сделать это довольно легко, но есть один интересный момент. Если вы читали предыдущую [часть](linux-bootstrap-3.md), то помните, что мы уже обновили сегментные регистры сразу после перехода в [защищённый режим](https://en.wikipedia.org/wiki/Protected_mode) в [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S). Так почему же нам снова нужно обновить значения в сегментных регистрах? Ответ прост. Ядро Linux также имеет 32-битный протокол загрузки и если загрузчик использует его для загрузки ядра, то весь код до `startup_32` будет пропущен. В этом случае `startup_32` будет первой точкой входа в ядро, и нет никаких гарантий, что сегментные регистры будут находиться в ожидаемом состоянии.

После того как мы проверили флаг `KEEP_SEGMENTS` и установили правильное значение в сегментные регистры, следующим шагом будет вычисление разницы между адресом, по которому мы загружены, и адресом, который был указан во время компиляции. Вы помните, что `setup.ld.S` содержит следующее определение в начале секции: `.head.text`: `. = 0`. Это значит, что код в этой секции скомпилирован для запуска по адресу `0`. Мы можем видеть это в выводе `objdump`:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

Утилита `objdump` говорит нам о том, что адрес `startup_32` равен `0`. Но на самом деле это не так. Наша текущая цель состоит в том, чтобы узнать настоящее местоположение. Довольно просто сделать это в [long mode](https://en.wikipedia.org/wiki/Long_mode), поскольку он поддерживает относительную адресацию с помощью указателя `rip`, но в настоящее время мы находимся в [защищённом режиме](https://en.wikipedia.org/wiki/Protected_mode). Для того чтобы узнать адрес `startup_32`, мы будем использовать общепринятый шаблон. Нам необходимо определить метку, перейти на эту метку и вытолкнуть вершину стека в регистр:

```assembly
call label
label: pop %reg
```

После этого регистр `%reg` будет содержать адрес метки. Давайте посмотрим на аналогичный код поиска адреса `startup_32` в ядре Linux:

```assembly
        leal	(BP_scratch+4)(%esi), %esp
        call	1f
1:      popl	%ebp
        subl	$1b, %ebp
```

Как вы помните из предыдущей части, регистр `esi` содержит адрес структуры [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L113), которая была заполнена до перехода в защищённый режим. Структура `boot_params` содержит специальное поле `scratch` со смещением `0x1e4`. Это 4 байтное поле будет временным стеком для инструкции `call`. Мы получаем адрес поля `scratch` + `4` байта и помещаем его в регистр `esp`. Мы добавили `4` байта к базовому адресу поля `BP_scratch`, поскольку поле является временным стеком, а стек на архитектуре `x86_64` растёт сверху вниз. Таким образом, наш указатель стека будет указывать на вершину стека. Далее мы видим наш шаблон, который я описал ранее. Мы переходим на метку `1f` и помещаем её адрес в регистр `ebp`, потому что после выполнения инструкции `call` на вершине стека находится адрес возврата. Теперь у нас есть адрес метки `1f` и мы легко сможем получить адрес `startup_32`. Нам просто нужно вычесть адрес метки из адреса, который мы получили из стека:

```
startup_32 (0x0)       +-----------------------+
                       |                       |
                       |                       |
                       |                       |
                       |                       |
                       |                       |
                       |                       |
                       |                       |
                       |                       |
1f (смещение 0x0 + 1f) +-----------------------+ %ebp - реальный физический адрес
                       |                       |
                       |                       |
                       +-----------------------+
```

`startup_32` слинкован для запуска по адресу `0x0` и это значит, что `1f` имеет адрес `0x0 + смещение 1f`, примерно `0x21` байт. Регистр `ebp` содержит реальный физический адрес метки `1f`. Таким образом, если вычесть `1f` из `ebp`, мы получим реальный физический адрес `startup_32`. В [протоколе загрузки ядра Linux](https://www.kernel.org/doc/Documentation/x86/boot.txt) описано, что базовый адрес ядра в защищённом режиме равен `0x100000`. Мы можем проверить это с помощью [gdb](https://en.wikipedia.org/wiki/GNU_Debugger). Давайте запустим отладчик и поставим точку останова на адресе `1f`, который равен `0x100021`. Если всё верно, то мы увидим `0x100021` в регистре `ebp`:

```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

Если мы выполним следующую инструкцию, `subl $1b, %ebp`, мы увидим следующее:

```
(gdb) nexti
...
...
...
ebp            0x100000	0x100000
...
...
...
```

Да, всё верно. Адрес `startup_32` равен `0x100000`. После того как мы узнали адрес метки `startup_32`, мы можем начать подготовку к переходу в [long mode](https://en.wikipedia.org/wiki/Long_mode). Наша следующая цель - настроить стек и убедится в том, что CPU поддерживает long mode и [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions).

Настройка стека и проверка CPU
--------------------------------------------------------------------------------

Мы не могли настроить стек, пока не знали адрес метки `startup_32`. Мы можем представить себе стек как массив, и регистр указателя стека `esp` должен указывать на конец этого массива. Конечно, мы можем определить массив в нашем коде, но мы должны знать его фактический адрес, чтобы правильно настроить указатель стека. Давайте посмотрим на код:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

Метка `boot_stack_end` определена в [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) и расположена в секции [.bss](https://en.wikipedia.org/wiki/.bss):

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

Прежде всего, мы помещаем адрес `boot_stack_end` в регистр `eax`, т.е регистр `eax` содержит адрес `0x0 + boot_stack_end`. Чтобы получить реальный адрес `boot_stack_end`, нам нужно добавить реальный адрес `startup_32`. Как вы помните, мы нашли этот адрес выше и поместили его в регистр `ebp`. В итоге регистр `eax` будет содержать реальный адрес `boot_stack_end` и нам просто нужно поместить его в указатель стека.

После того как мы создали стек, следующим шагом является проверка CPU. Так как мы собираемся перейти в `long mode`, нам необходимо проверить, поддерживает ли CPU `long mode` и `SSE`. Мы будем делать это с помощью вызова функции `verify_cpu`:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

Она определена в [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/kernel/verify_cpu.S) и содержит пару вызовов инструкции [CPUID](https://en.wikipedia.org/wiki/CPUID). Данная инструкция
используется для получения информации о процессоре. В нашем случае она проверяет поддержку `long mode` и `SSE` и с помощью регистра `eax` возвращает `0` в случае успеха или `1` в случае неудачи.

Если значение `eax` не равно нулю, то мы переходим на метку `no_longmode`, которая останавливает CPU вызовом инструкции `hlt` до тех пор, пока не произойдёт аппаратное прерывание:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

Если значение `eax` равно нулю, то всё в порядке и мы можем продолжить.

Расчёт адреса релокации
--------------------------------------------------------------------------------

Следующим шагом является вычисление адреса релокации для декомпрессии, если это необходимо. Мы уже знаем, что базовый адрес 32-битной точки входа в ядро Linux - `0x100000`, но это 32-битная точка входа. Базовый адрес ядра по умолчанию определяется значением параметра конфигурации ядра `CONFIG_PHYSICAL_START`. Его значение по умолчанию `0x1000000` или `16 Мб`. Основная проблема заключается в том, что если происходит краш ядра, разработчик должен иметь `rescue ядро` ("спасательное" ядро) для [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt), которое сконфигурировано для загрузки из другого
адреса. Для решения этой проблемы ядро Linux предоставляет специальный параметр конфигурации - `CONFIG_RELOCATABLE`. Как вы можете прочесть в документации ядра:

```
Это создает образ ядра, который сохраняет информацию о релокации
поэтому он может быть загружен где-либо, кроме стандартного 1 Мб.

Примечание: Если CONFIG_RELOCATABLE=y, то ядро запускается с адреса,
на который он был загружен, а физический адрес времени компиляции
(CONFIG_PHYSICAL_START) используется как минимальная локация.
```

Проще говоря, это означает, что ядро с той же конфигурацией может загружаться с разных адресов. С технической точки зрения это делается путём компиляции декомпрессора как [адресно-независимого кода](https://en.wikipedia.org/wiki/Position-independent_code). Если мы посмотрим на [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile), то мы увидим, что декомпрессор действительно скомпилирован с флагом `-fPIC`:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

Когда мы используем адресно-независимый код, адрес получается путём добавления адресного поля инструкции и значения счётчика команд программы. Код, использующий подобную адресацию, возможно загрузить с любого адреса. Вот почему мы должны были получить реальный физический адрес `startup_32`. Давайте вернёмся к коду ядра Linux. Наша текущая цель состоит в том, чтобы вычислить адрес, на который мы можем переместить ядро для декомпрессии. Расчёт этого адреса зависит от параметра конфигурации ядра `CONFIG_RELOCATABLE`. Давайте посмотрим на код:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
```

Следует помнить, что регистр `ebp` содержит физический адрес метки `startup_32`. Если параметр `CONFIG_RELOCATABLE` включён во время конфигурации ядра, то мы помещаем этот адрес в регистр `ebx`, выравниваем по границе, кратной `2 Мб` и сравниваем его со значением `LOAD_PHYSICAL_ADDR`. `LOAD_PHYSICAL_ADDR` является макросом, определённым в [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/boot.h) и выглядит следующим образом:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

Как мы можем видеть, он просто расширяет адрес до значения выравнивания `CONFIG_PHYSICAL_ALIGN` и представляет собой физический адрес, по которому будет загружено ядро. После сравнения `LOAD_PHYSICAL_ADDR` и значения регистра `ebx`, мы добавляем смещение от `startup_32`, по которому будет происходить декомпрессия образа ядра. Если во время компиляции параметр `CONFIG_RELOCATABLE` не включён, мы просто помещаем адрес по умолчанию и добавляем к нему `z_extract_offset`.

После всех расчётов у нас в распоряжении `ebp`, содержащий адрес, по которому будет происходить загрузка, и `ebx`, содержащий адрес, по которому ядро будет перемещено после декомпрессии. Но это еще не конец. Сжатый образ ядра должен быть перемещён в конец буфера декомпрессии, чтобы упростить вычисления местоположения, по которому ядро будет расположено позже:

```assembly
1:
    movl	BP_init_size(%esi), %eax
    subl	$_end, %eax
    addl	%eax, %ebx
```

мы помещаем значение из `boot_params.BP_init_size` (или значение заголовка настройки ядра из `hdr.init_size`) в регистр `eax`. `BP_init_size` содержит наибольшее значение между сжатым и распакованным [vmlinux](https://en.wikipedia.org/wiki/Vmlinux). Затем мы вычитаем адрес символа `_end` из этого значения и добавляем результат вычитания в регистр `ebx`, который хранит базовый адрес для декомпрессии ядра.

Подготовка перед входом в long mode
--------------------------------------------------------------------------------

Теперь, когда у нас есть базовый адрес, на который мы будем перемещать сжатое ядро, нам необходимо сделать последний шаг, прежде чем мы сможем перейти в 64-битный режим. Во-первых, нам необходимо обновить [глобальную таблицу дескрипторов](https://en.wikipedia.org/wiki/Global_Descriptor_Table) с 64-битными сегментами, потому что перемещаемое ядро может быть запущено по любому адресу ниже 512 Гб:

```assembly
	addl	%ebp, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

Здесь мы настраиваем базовый адрес `глобальной таблицы дескрипторов` на адрес, где мы фактически загружены, и загружаем таблицу с помощью инструкции `lgdt`.

Чтобы понять магию смещений `gdt`, нам нужно взглянуть на определение `глобальной таблицы дескрипторов`. Мы можем найти его определение в том же [файле](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) исходного кода:

```assembly
	.data
gdt64:
	.word	gdt_end - gdt
	.long	0
	.word	0
	.quad   0
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS *//
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* Дескриптор TS */
	.quad   0x0000000000000000	/* Продолжение TS */
gdt_end:
```

Мы видим, что она расположена в секции `.data` и содержит пять дескрипторов: `32-битный` дескриптор для сегмента кода ядра, `64-битный` сегмент ядра, сегмент данных ядра и два дескриптора задач.

Мы уже загрузили `глобальную таблицу дескрипторов` в предыдущей [части](linux-bootstrap-3.md), и теперь мы делаем почти то же самое здесь, но теперь дескрипторы с `CS.L = 1` и `CS.D = 0` для выполнения в 64-битном режиме. Как мы видим, определение `gdt` начинается с двух байт: `gdt_end - gdt`, который представляет последний байт `gdt` или лимит таблицы. Следующие 4 байта содержат базовый адрес `gdt`.

После того как `глобальная таблица дескрипторов` загружена с помощью инструкции `lgdt`, нам необходимо включить режим [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension), поместив значение регистра `cr4` в `eax`, установить в нём пятый бит и загрузить его снова в `cr4`:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Мы почти закончили все подготовки перед входом в 64-битный режим. Последний шаг заключается в создании таблицы страниц, но прежде чем сделать это, необходимо рассказать о long mode

Long mode
--------------------------------------------------------------------------------

[Long mode](https://en.wikipedia.org/wiki/Long_mode) - нативный режим для процессоров [x86_64](https://en.wikipedia.org/wiki/X86-64). Прежде всего посмотрим на некоторые различия между `x86_64` и `x86`.

`64-битный` режим предоставляет следующие особенности:

* 8 новых регистров общего назначения с `r8` по `r15` + все регистры общего назначения теперь 64-битные;
* 64-битный указатель инструкции - `RIP`;
* Новый режим работы - Long mode;
* 64-битные адреса и операнды;
* Относительная адресация RIP (мы увидим пример этого в следующих частях).

Long mode является расширением унаследованного защищённого режима. Он состоит из двух подрежимов:

* 64-битный режим;
* режим совместимости.

Для переключения в `64-битный` режим необходимо сделать следующее:

* Включить [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
* Создать таблицу страниц и загрузить адрес таблицы страниц верхнего уровня в регистр `cr3`;
* Включить `EFER.LME`;
* Включить страничную организацию памяти.

Мы уже включили `PAE` путём установки бита `PAE` в регистре управления `cr4`. Наша следующая цель - создать структуру для [страничной организации](https://en.wikipedia.org/wiki/Paging). Мы увидим это в следующем параграфе.

Ранняя инициализация таблицы страниц
--------------------------------------------------------------------------------

Итак, мы уже знаем, что прежде чем мы сможем перейти в `64-битный` режим, необходимо создать таблицу страниц. Давайте посмотри на создание ранних `4 гигабайтных` загрузочных таблиц страниц.

**ПРИМЕЧАНИЕ: я не буду описывать теорию виртуальной памяти. Если вам необходимо больше информации по виртуальной памяти, см. ссылки в конце этой части.**

Ядро Linux использует `4 уровневую` страничную организацию, и в целом мы создадим 6 таблиц страниц:

* Одну таблицу `PML4 (карта страниц 4 уровня, Page Map Level 4)` с одной записью;
* Одну таблицу `PDP (указатель директорий страниц, Page Directory Pointer)` с четырьмя записями;
* Четыре таблицы директорий страниц с `2048` записями.

Давайте посмотрим на реализацию. Прежде всего, мы очищаем буфер для таблиц страниц в памяти. Каждая таблица имеет размер в `4096` байт, поэтому нам необходимо очистить `24` Кб буфера:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl
```

Мы помещаем адрес `pgtable + ebx` (вы помните, что `ebx` содержит адрес, по которому ядро будет перемещено после декомпрессии) в регистр `edi`, очищаем регистр `eax` и устанавливаем регистр `ecx` в `6144`.

Инструкция `rep stosl` записывает значение `eax` в `edi`, увеличивает значение в регистре `edi` на `4` и уменьшает значение в регистре `ecx` на `1`. Эта операция будет повторятся до тех пор, пока значение регистра `ecx` больше нуля. Вот почему мы установили `ecx` в `6144` (или `BOOT_INIT_PGT_SIZE/4`).

Структура `pgtable` определена в конце файла [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S):

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill BOOT_PGT_SIZE, 1, 0
```

Как мы видим, она находится в секции `.pgtable`  и его размер зависит от опции конфигурации ядра `CONFIG_X86_VERBOSE_BOOTUP`:

```C
#  ifdef CONFIG_X86_VERBOSE_BOOTUP
#   define BOOT_PGT_SIZE	(19*4096)
#  else /* !CONFIG_X86_VERBOSE_BOOTUP */
#   define BOOT_PGT_SIZE	(17*4096)
#  endif
# else /* !CONFIG_RANDOMIZE_BASE */
#  define BOOT_PGT_SIZE		BOOT_INIT_PGT_SIZE
# endif
```

После того как мы получили буфер для `pgtable`, мы можем начать с создания таблицы страниц верхнего уровня - `PML4` - следующим образом:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Здесь мы снова помещаем относительный адрес `pgtable` в `ebx` или, другими словами, относительный адрес `startup_32` в регистр `edi`. Далее мы помещаем этот адрес со смещением `0x1007` в регистр `eax`. Смещение `0x1007` равно `4096` байтам, которые представляют собой размер `PML4` плюс `7`. `7` здесь представляет флаги `PML4`. В нашем случае это флаги `PRESENT+RW+USER`. В конечном счёте мы просто записали адрес первого элемента `PDP` в `PML4`.

Следующий шаг - создание четырёх записей `директории страниц` в таблице `указателя директорий страниц` с теми же флагами `PRESENT+RW+USE`:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Мы помещаем базовый адрес указателя директорий страниц, который равен `4096` или, другими словами, смещение `0x1000` от таблицы `pgtable` в `edi`, и адрес первой записи указателя директорий страниц в регистр `eax`. Значение `4`, помещённое в регистр `ecx`, будет счётчиком в следующем цикле, в котором мы записываем адрес первой записи таблицы указателя директорий страниц в регистр `edi`. После этого `edi` будет содержать адрес первой записи указателя директорий страниц с флагами `0x7`. Далее мы просто вычисляем адрес следующих записей указателя директорий страниц, где каждая запись имеет размер `8` байт, и записываем их адреса в `eax`. Последний шаг в создании страничной организации памяти - создание `2048` записей с `2 мегабайтными` страницами:

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Здесь мы делаем почти тоже самое, как и в предыдущем примере; все записи с флагами `$0x00000183`: `PRESENT + WRITE + MBZ`. В итоге мы будем иметь `2048` `2 мегабайтных` страниц:

```python
>>> 2048 * 0x00200000
4294967296
```

или `4 гигабайтную` таблицу страниц. Мы закончили создание нашей ранней структуры таблицы страниц, которая отображает `4` Гб на память и теперь мы можем поместить адрес таблицы страниц верхнего уровня - `PML4` - в регистр управления `cr3`:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

На этом всё. Все подготовки завершены и теперь мы можем перейти в long mode.

Переход в 64-битный режим
--------------------------------------------------------------------------------

В первую очередь нам нужно установить флаг `EFER.LME` в [MSR](http://en.wikipedia.org/wiki/Model-specific_register), равный `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Здесь мы помещаем флаг `MSR_EFER` (который определён в [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/msr-index.h#L7)) в регистр `ecx` и вызываем инструкцию `rdmsr`, которая считывает регистр [MSR](http://en.wikipedia.org/wiki/Model-specific_register). После выполнения `rdmsr`, полученные данные будут находится в `edx:eax`, которые будут зависеть от значения `ecx`. Далее мы проверяем бит `EFER_LME` инструкцией `btsl` и с помощью инструкции `wrmsr` записываем данные из `eax` в регистр `MSR`.

На следующем шаге мы помещаем адрес сегмента кода ядра в стек (мы определили его в GDT) и помещаем адрес функции `startup_64` в `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

После этого мы помещаем адрес в стек и включаем поддержку страничной организации путём установки битов `PG` и `PE` в регистре `cr0`:

```assembly
    pushl	%eax
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

и выполняем инструкцию:

```assembly
lret
```

Вы должны помнить, что на предыдущем шаге мы поместили адрес функции `startup_64` в стек, и после инструкции `lret`, CPU извлекает адрес и переходит по нему.

После всего этого, мы, наконец, в 64-битном режиме:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

На этом всё!

Заключение
--------------------------------------------------------------------------------

Это конец четвёртой части о процессе загрузки ядра Linux. В следующей части мы увидим декомпрессию ядра и многое другое.

**От переводчика: пожалуйста, имейте в виду, что английский - не мой родной язык, и я очень извиняюсь за возможные неудобства. Если вы найдёте какие-либо ошибки или неточности в переводе, пожалуйста, пришлите pull request в [linux-insides-ru](https://github.com/proninyaroslav/linux-insides-ru).**

Ссылки
--------------------------------------------------------------------------------

* [Защищённый режим](http://en.wikipedia.org/wiki/Protected_mode)
* [Документация для разработчиков ПО на архитектуре Intel® 64 и IA-32](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU компоновщик](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Страничная организация памяти (Википедия)](http://en.wikipedia.org/wiki/Paging)
* [Моделезависимый регистр](http://en.wikipedia.org/wiki/Model-specific_register)
* [Инструкция .fill](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Предыдущая часть](linux-bootstrap-3.md)
* [Страничная организация памяти (OSDEV)](http://wiki.osdev.org/Paging)
* [Системы страничной организации памяти](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [Пособие по страничной организации на x86](http://www.cirosantilli.com/x86-paging/)
