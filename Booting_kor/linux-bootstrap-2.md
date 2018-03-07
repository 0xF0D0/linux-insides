커널 부팅 과정 파트2
================================================================================

커널 setup의 첫 과정
--------------------------------------------------------------------------------

우리는 지난 [파트](linux-bootstrap-1.md)에서 리눅스 커널을 파기시작했고, 커널 setup코드의 첫부분을 보았다. [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c) 에서 첫 `main`함수의 첫 c 코드까지 점프하는것을 보았다.

이 파트에선, 커널 setup 단계에서 다음을 살펴볼 것이다.
* `protected mode`가 무엇인지,
* `protected mode`로의 전환,
* Heap과 콘솔 초기화,
* 메모리 detection, CPU 활성화와 키보드 초기화
* 그리고 더많은 것들.


Protected mode
--------------------------------------------------------------------------------

Intel 64 [Long Mode](http://en.wikipedia.org/wiki/Long_mode)로 가기전에, 커널은 CPU를 protected 모드로 바꿔주어야 한다.

[protected mode](https://en.wikipedia.org/wiki/Protected_mode)는 무엇일까? Protected mode 는 1982년에 처음으로 x86 구조에 추가되었고, [80286](http://en.wikipedia.org/wiki/Intel_80286) 프로세서부터 Intel 64와 long mode가 나오기 전까지 Intel 프로세서의 메인 모드였다.

[Real mode](http://wiki.osdev.org/Real_Mode)에서 넘어오게된 이유는 RAM을 매우 제한적으로밖에 사용할 수 없기 떄문이다. 지난 파트에서 봤듯이 2<sup>20</sup> 바이트(1MB)나 혹은 특정 경우엔 640KB만큼의 RAM만 Real mode에서 사용할 수 있다.

Protected mode는 많은 변화를 가져왔다, 그중 주된 변화는 메모리 관리방식이다. 20-bit 주소 버스는 32-bit 주소 bus로 바뀌었다. 이는 메모리를 4GB까지 사용 할 수 있게 만들었다. 또한 [paging](http://en.wikipedia.org/wiki/Paging)지원이 추가되었다, 이게 무엇인지는 다음 섹션에서 볼 수 있다.

Protected mode에서의 메모리 관리는 두가지로 나뉜다:

* Segmentation
* Paging

여기선 segmentation에 대해서만 알아볼 것이다. Paging은 다음 섹션에서 알게 될것이다.

전 파트에서 보았듯이, real mode의 주소는 두 파트로 구성된다:

* Segment의 base address
* Segment base로 부터의 offset

이 두가지를 알고있으면 다음과 같이 주소를 구한다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

메모리 segmentation은 protected mode에서 완전히 다시 구현되었다. 이제 64KB 자리 고정segment는 없다. 그 대신, _Segment Descriptor_라는 데이터구조에 segment의 크기와 위치가 정의 되어있다. 그리고 segment descriptor는 `Global Descriptor Table` (GDT)라는 데이터 구조에 저장된다.

GDT는 메모리에 상주하는 구조이다. 메모리에 고정된 장소가 없기때문에, `GDTR`이라는 특별한 레지스터에 주소가 저장되어있다. 나중에 우리는 GDT가 어떻게 리눅스 커널에서 로드되는지 볼 것이다. 다음 명령어처럼 메모리에 로딩을 한다:

```assembly
lgdt gdt
```

여기서 `lgdt`명령어는 base 주소와, global descriptor table의 limit(크기)를 `GDTR`레지스터에 로드한다. `GDTR` 은 48-bit 레지스터이고 두 파트로 구성되어 있다:

 * Global descriptor table의 크기 (16 bit);
 * Global descriptor table의 주소 (32 bit).

위에서 언급했듯이 GDT는 메모리 segments를 나타내는 `segment descriptor`들을 갖고있는 테이블이다. 각 descriptor들은 64-bit이며 구조는 다음과 같다:

```
 63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|     |     | D   |     | L   | 19:16 |     |     |     | 1   | C   | R   | A   |     |
| --- |

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|     |     |
| --- |
```

Real mode에 비해서 복잡해보이지만, 알고보면 쉬우니 걱정하지마라. 예를들어 descriptor 시작점에있는 LIMIT 15:0의 의미는 Limit의 0~15까지의 bit를 의미한다. 나머지 LIMIT은 descriptor의 48-51bit에 있는 LIMIT 19:16에 존재한다. 따라서 Limit의 크기는 0:19 즉, 20bit이 된다. 더 자세히 살펴보자:

1. Limit[20-bit]는 비트범위 0-15와 48-51로 나뉘어져 있다. Limit은 (세그먼트의 크기-1)을 나타내는데, 계산은 `G`(Granularity) bit에 따라서 달라진다.

  * `G` (bit 55) 가 0이고 segment limit이  0이면, segment의 크기는 1 Byte이다
  * `G` 가 1이고 segment limit이 0이면, segment의 크기는 4096 Byte이다
  * `G` 가 0이고 segment limit이 0xfffff이면, segment의 크기는 1 Megabyte이다
  * `G` 가 1이고 segment limit이 0xfffff이면, segment의 크기는 4 Gigabyte이다

  자, 해석을 해보면
  * G가 0일땐, Limit의 단위는 1 Byte 이고 segment의 최대 크기는 1 Megabyte이다.
  * G가 1일떈, Limit의 단위는 4096 Bytes = 4 KBytes = 1 Page 이고 segment의 최대 크기는 4 Gigabyte 이다. 실제론, G가 1일때, Limit 은 왼쪽으로 12만큼 shift된다. 즉 20 bit + 12 bit = 32 bit이 되고 2<sup>32</sup> = 4 Gigabyte가 된다.

2. Base[32-bit]는 비트범위 16-31, 32-39, 56-63로 나뉘어져 있다. Base는 segment의 시작 물리주소를 나타낸다.

3. Type/Attribute[5-bits]는 비트범위 40-44에 있다. Segment의 타입과 접근제어를 정의한다.
  * `S` flag는 비트 44에 있으며, descriptor의 타입을 정의한다. `S`가 0이면 segment는 system segment이다, `S`가 1이면 code나 data segment이다. (Stack segment는 read/write가 가능한 data segment이다).

Segment가 code인지 data인지 판별하기 위해선, Ex(43 bit)를 확인해 봐야한다. 0일경우 data, 1일경우 code segment이다.

Segment는 다음타입들중 하나가 된다:

```
| Type Field                  | Descriptor Type | Description                        |
| --------------------------- | --------------- | ---------------------------------- |
| Decimal                     |                 |
| 0    E    W   A             |                 |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
| C    R   A                  |                 |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
```

위에서 봤듯이 첫번째 비트(43 bit)이 `0`이면 _data_ segment이고 `1`이면 _code_ segment이다. 다음 세비트(40, 41, 42)는 `EWA`(*E*xpansion *W*ritable *A*ccessible) 혹은 CRA(*C*onforming *R*eadable *A*ccessible)이다.
  * E(bit 42)가 0이면, expand up이고, 아니면, expand down이다. 자세한 정보는 [여기서](http://www.sudleyplace.com/dpmione/expanddown.html).
  * W(bit 41)(Data Segment 용)이 1이면, 쓰기 권한이 추가된다, 0일경우, segment는 read-only이다. Data segment에서 읽기 권한은 항상 부여된다는 것에 주목하라.
  * A(bit 40)는 segment가 processor에 의해 접근 될 수 있는지 없는지를 나타낸다.
  * C(bit 43)는 conforming bit(code selector 용)이다. C가 1이면, segment code는 하위 privilege 레벨(eg. 유저)에서 실행 될 수 있다. 만약 C가 0 이면, 같은 previlege 레벨에서만 실행 될 수 있다.
  * R(bit 41) 은 코드세그먼트의 읽기 권한이다; 1일 경우 해당 코드 세그먼트를 읽을 수 있다, 코드세그먼트에서 쓰기 권한은 절대 허락되지 않는다.

4. DPL[2-bits] (Descriptor Privilege Level) 은 45-46bit 에 있다. 해당 세그먼트의 previlege level을 나타낸다. Previlege level은 0-3까지 이며 3이 가장 권한이 높다.

5. P flag(bit 47) 는 세그먼트가 메모리에 올라와있는지를 판단한다. 만약 0이면, 세그먼트는 _invalid_로 표시되며 프로세서는 해당 세그먼트 읽기를 거부할 것이다.

6. AVL flag(bit 52) - Available and reserved bits. Linux에선 무시된다.

7. The L flag(bit 53) 코드 세그먼트가 64bit code를 포함하는지 알려준다. Set되어있다면 코드 세그먼트는 64-bit 모드에서 동작한다.

8. The D/B flag(bit 54)  (Default/Big flag) 는 오퍼랜드(피연산자)의 사이즈를 결정한다. 1이라면 32bit, 아니라면 16bit이다.

세그먼트 레지스터(eg. cs,ds)들은 real mode와 마찬가지로 segment selector를 갖는다. 다만 protected mode에선 다르게 동작한다. 각 segment descriptor들은 대응되는 16-bit 구조 segment selector들을 갖고 있다.

```
 15             3 2  1     0
-----------------------------
| Index | TI  | RPL |
| ----- |
```

살펴보면,
* **Index** 는 GDT에서 descriptor의 인덱스를 나타낸다.
* **TI**(Table Indicator) 는 어디서 descriptor를 찾을지를 나타낸다. 0이라면 GDT를 살펴보고 아니라면 LDT(Local Descriptor Table)을 살펴본다.
* And **RPL** 는 요청자의 prevelige level을 나타낸다..

모든 segment register는 Visible 부분과 Hidden 부분으로 나뉜다.
* Visible - Segment Selector가 여기 저장된다.
* Hidden -  Segment Descriptor (base, limit, attributes & flags등 으로 구성) 이 여기 저장된다.

Protected mode에서 물리주소를 얻기 위해 다음과정을 거친다:

* segment selector가 segment register에 로드 된다.
* CPU는 segment selector의 `GDT 주소 + Index`오프셋을 통해 segment descriptor를 찾고 segment register의 hidden부분에 descriptor를 로드한다.
* Paging이 꺼져있다면, segment의 물리주소는 다음 공식으로 구해진다: Base address (전 단계의 descriptor에서 구한 Base주소) + Offset.

도식화하면 다음과 같다:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

Real mode에서 protected mode로의 전환 알고리즘은 다음과 같다:

* 인터럽트 끄기
* GDT를 `lgdt`명령어로 로드
* CR0(Control Register 0)의 PE (Protection Enable)를 1로 설정
* Protected mode code로 점프

다음파트에서 우리는 linux커널이 protected mode로 점프하는 전체 과정을 볼 것이다, 그전에 우리는 준비과정을 살펴보아야한다.

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)를 보면 키보드 초기화, heap 초기화 등등이 보인다. 자세히 살펴보자.

"zeropage"로 부트 파라미터 복사
--------------------------------------------------------------------------------

"main.c"의 `main`루틴에서 시작해보자. `main`에서 제일 처음 불리는 함수는 [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L30)이다. 이 함수는 [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/bootparam.h#L113)에 정의 되어 있는 `boot_params`구조에 상응하는 필드에 kernel setup 헤더를 복사한다.

`boot_params` 구조에는 `struct setup_header hdr` 필드가 있다. 이구조에는 [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에 정의된것가 동일한 필드가 존재하고 kernel compile/build time에 부트로더에 의해 채워진다. `copy_boot_params` 은 두가지 일을 한다:

1. [header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L281)에서 `hdr`을 복사해서 `setup_header`필드의 `boot_params` structure에 붙여 넣는다.

2. Old command line protocol로 커널이 로드되었다면 커널 commaned line의 포인터를 업데이트한다.

`hdr`가 [copy.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/copy.S)에 정의된 `memcpy` 함수에 의해 복사된다:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

첫째로, `memcpy`나 여기서 정의된 다른 루틴들을 보면 `GLOBAL`과 `ENDPROC`매크로로 시작하고 끝나는 것을 볼 수 있다매크로로 시작하고 끝나는 것을 볼 수 있다. `GLOBAL`은 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/linkage.h)에 있고, `globl` 지시어와 라벨이 정의되어 있다. `ENDPROC`은 같은 파일에 존재하며 `name` 을 함수이름으로 정의하고 `name`의 사이즈로 끝난다.

`memcpy`의 구현은 간단하다. 일단 `si`와 `di` 레지스터값은 실행중에 바뀌기 때문에 이를 보존하기 위해 스택에 push한다. `memcpy`나 copy.S의 다른 함수들은 `fastcall` 호출 규약(calling convention)을 사용한다. 따라서 함수 파라미터들은 `ax`, `dx` 그리고 `cx`레지스터에 저장된다. `memcpy`를 호출하는것은 다음과 같다:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

따라서,
* `ax` 에는 `boot_params.hdr`의 주소가 있다.
* `dx` 에는 `hdr`의 주소가 있다.
* `cx` `hdr`의 크기(바이트)가 있다.

`memcpy`는 `boot_params.hdr`의 주소를 `di`로 넣고 `cx`를 스택에 저장한다. 이후 `cx`값을 오른쪽으로 2만큼 shift(4로 나누기)하고 `si`의 주소로부터 4바이트만큼 복사하여 `di`의 주소값에 쓴다. 위의 반복 작업이 끝나면 다시 `hdr`의 크기를 가져와서 4바이트로 정렬한다. 만약 4바이트로 정렬되지못한 나머지 바이트가 있다면 마저 복사한다. 마지막으로 `si`와 `di`값을 스택으로부터 복원하고 Copy작업은 끝이다.

Console 초기화
--------------------------------------------------------------------------------

`hdr`가 `boot_params.hdr`으로 복사되고 나면, 다음 작업은 [lrch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/early_serial_console.c)에 정의되어있는 `console_init`함수를 사용하여 콘솔을 초기화하는 것이다. 

`console_init`함수는 커맨드라인의 `earlyprintk`옵션을 찾아보고, search가 성공했다면, 해당 직렬포트의 주소와 baud rate를 받아오고 직렬포트를 초기화한다. `earlyprintk` 커맨드라인 옵션은 다음중 하나와 같다:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

직렬포트 초기화후에 우리는 다음과 같은 출력을 볼 수 있다:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

`puts`는 [tty.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/tty.c)에 정의되어 있다. 링크에서 볼 수 있듯이 반복문을 돌면서 캐릭터마다 `putchar`함수를 호출하고있다. 구현은 다음과 같다:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))`는 이코드가 `.inittext` 섹션이 위치한다는 것을 뜻한다. [setup.ld](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L19) 링커 파일을 보면 확인 할 수 있다.

일단, `putchar`는 `\n`이 있는지 체크하고, 만약 있다면, `\r`을 먼저 출력한다. 그후 BIOS `0x10` 인터럽트를 이용해 VGA 화면에 문자를 출력한다.

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

여기서 `initregs`는 `biosregs` structure를 받아서 `biosregs`를 `memset`함수를 사용해서 0으로 채운다. 그후 register값들을 채운다.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

[memset](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/copy.S#L36)의 구현은 다음과 같다:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

`memcpy`함수처럼 `memset`함수도 `fastcall` 호출 규약을 사용한다. 따라서 함수의 파라미터는 `ax`, `dx`, `cx`레지스터에 저장된다.

`memset`의 구현은 memcpy와 비슷하다. `di`레지스터를 스택에 저장하고 
The implementation of `memset` is similar to that of memcpy. It saves the value of the `di` register on the stack and puts the value of`ax`, which stores the address of the `biosregs` structure, into `di` . Next is the `movzbl` instruction, which copies the value of `dl` to the lower 2 bytes of the `eax` register. The remaining 2 high bytes  of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, if we need to fill a structure whose size is 4 bytes with the value `0x7` with memset, `eax` will contain the `0x00000007`. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses the `rep; stosl` instruction to copy `eax` into `es:di`.

The rest of the `memset` function does almost the same thing as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/tty.c#L30) and `inb/outb` instructions if it was set.

Heap initialization
--------------------------------------------------------------------------------

After the stack and bss section have been prepared in [header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L116) with the [`init_heap`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L116) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/bootparam.h#L22) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321) structure in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

which means `heap_end_ptr` or `_end` + `512` (`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see what it is used for, how to use it and how it is implemented in the next posts.

CPU validation
--------------------------------------------------------------------------------

The next step as we can see is cpu validation through the `validate_cpu` function from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/cpu.c).

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/cpucheck.c#L112) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
`check_cpu` checks the CPU's flags, the presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in the case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparations for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

Memory detection
--------------------------------------------------------------------------------

The next step is memory detection through the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the CPU. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of the **0xE820** interface here.

Let's look at the implementation of the `detect_memory_e820` function from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` contains the size of the buffer which will contain data about the memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts with a call to the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, this function collects data from the address allocation table and writes this data into the `e820_entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (whether the particular segment is usable or reserved)

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Next, we may see a call to the `set_bios_mode` function. As we may see, this function is implemented only for the `x86_64` mode:

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

The `set_bios_mode` function executes the `0x15` BIOS interrupt to tell the BIOS that [long mode](https://en.wikipedia.org/wiki/Long_mode) (if `bx == 2`) will be used.

Keyboard initialization
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with a call to the [`keyboard_init()`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function. It then calls the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt to query the status of the keyboard.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set the repeat rate and delay.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries but we will get back to them in later parts. Let's take a short look at these functions:

The first step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. It checks the CPU level and if it is correct, calls `0x15` to get the info and saves the result to `boot_params`.

Next, the [query_apm_bios](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After `0x15` finishes executing, the `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), the carry flag (it must be 0 if `APM` supported) and the value of the `cx` register (if it's 0x02, the protected mode interface is supported).

Next, it calls `0x15` again, but with `ax = 0x5304` to disconnect the `APM` interface and connect the 32-bit protected mode interface. In the end, it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if the `CONFIG_APM` or `CONFIG_APM_MODULE` compile time flag was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look at how `query_edd` is implemented.

First of all, it reads the [edd](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

where `0x80` is the first hard drive and the value of the `EDD_MBR_SIG_MAX` macro is 16. It collects data into an array of [edd_info](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about the insides of the Linux kernel. In the next part, we will see video mode setting and the rest of the preparations before the transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
