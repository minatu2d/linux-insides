Quá trinhg boot nhân. Phần 2.
================================================================================

Các bước đầu tiên khi thiết lập nhân
--------------------------------------------------------------------------------

 Chúng ta đã bắt đầu tìm hiểu bên trong nhân Linux ở chương trước [part](linux-bootstrap-1.md) đã xem phần đầu trong đoạn code thiết lập nhân (kernel setup code). Chúng ta đã dừng lại đoạn gọi đến hàm `main` (là hàm đầu tiên được viết bằng C) trong [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). 

Trong chương này, chúng ta vẫn tiếp tục nghiên cứu thêm về đoạn code setup nhân 
* Xem cái `protected mode` là cái gì,
* Các chuẩn bị để chuyển sang mode đó,
* Khởi tạo bộ nhớ heap, console,
* Phát hiện bộ nhớ (memory detection), kiểm tra CPU (cpu validation), khởi tạo bàn phím 
* và nhiều, nhiều hơn thế.

 Nào, hãy bắt đầu nào.

 Chế độ Protected mode (Protected mode)
--------------------------------------------------------------------------------

 Trước khi bạn có thể chuyển sang Intel64 native (native Intel64) [Long Mode](http://en.wikipedia.org/wiki/Long_mode), kernel phải chuyển CPU sang chế độ protected mode.

Chế độ [protected mode](https://en.wikipedia.org/wiki/Protected_mode) là cái gì? Chế độ Protected mode ban đầu được thêm vào kiến trúc x86 vào năm 1982 và là chế độ chính cho bộ xử lý Intel từ [80286](http://en.wikipedia.org/wiki/Intel_80286) cho đến Intel 64 và long mode mới được sinh ra.

 Lý do chính để chuyển sang từ chế độ [Real mode](http://wiki.osdev.org/Real_Mode) là do sự hạn chế về kích thước bộ nhớ có thể truy cập. Bạn có thể nhớ một chút về chương trước, chỉ có 2<sup>20</sup> bytes hay 1 Megabyte, hay thâm chí chỉ 640 Kilobytes của RAM có thể sử dụng trong chế độ Real Mode.

Chế độ Protected mode mang đến rất nhiều thay đổi, nhưng thay đổi chính là trong quản lý bộ nhớ. Bus địa chỉ 20-bit được thay thế bằng bus địa chỉ 32-bit. Nó cho phép truy cập đến 4 Gigabytes bộ nhớ so với 1 Megabyte như trong chế độ Real mode. Cả việc hỗ trợ phân trang [paging](http://en.wikipedia.org/wiki/Paging) cũng được đưa vào, bạn có thể độc về nó ở trong chương sau.

 Quản lý bộ nhớ trong chế độ Protected Mode được chia làm 2 phần, hầu như tách biệt nhau:

* Segmentation (phân đoạn)
* Paging (phân trang)

 Trong chương này chúng ta chỉ nói về phân đoạn (segmentation). Phân trang (Paging) được miêu tả trong chương sau. 

 Như đã nói ở chương trước, mỗi địa chỉ trong real mode chứa 2 phần:

* Địa chỉ cơ bản của đoạn (Base address of the segment)
* Khoảng cách tính từ đoạn cơ bản (Offset from the segment base)

 Chúng ta có thể biết địa chỉ thực nếu biết 2 thành phần này theo công thức sau:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Đoạn bộ nhớ (Memory segmentation) được làm lại hoàn toàn trong chế độ protected mode. Không có các đoạn với kích thước cố định 64 Kilobyte nữa. Thay vào đó, kích thước và vị trí mỗi đoạn được biểu diễn bằng một cấu trúc tương ứng có tên _Segment Descriptor_. Các miêu tả đoạn này lại được lưu trong một cấu trúc khác gọi là `Global Descriptor Table` (GDT).

 Cái GDT là một cấu trúc nằm suốt trên bộ nhớ. Nó không có vị trí cố định trên bộ nhớ, vì thế địa chỉ cơ bản được lưu ở vị trí thanh ghi `GDTR`. Ở phần sau, chúng ta sẽ xem đoạn load giá trị cho GDT (GDT loading) trong code của nhân Linux. Sẽ có một thao tác load GDT vào bộ nhớ kiểu như sau:

```assembly
lgdt gdt
```

Lệnh `lgdt` sẽ load địa chỉ cơ bản và giới hạn bộ nhớ của GDT (global descriptor table) vào thanh ghi `GDTR`. `GDTR` là một thanh ghi 48-bit, chứa 2 phần:

 * kích thước (16-bit) của GDT (global descriptor table);
 * địa chỉ (32-bit) của GDT đó (global descriptor table).

 Như đã nói đến trước đó GDT chứa một miêu tả đoạn hay `segment descriptors` miêu tả các đoạn trong bộ nhớ. Mỗi miêu tả này sẽ miêu tả kích thước trong 64-bits. Cấu trúc chung của mô tả này như sau:

```
31          24        19      16              7            0
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           | 0
|                             |                            |
------------------------------------------------------------
```

Trong có vẻ đáng sợ phải không so với chế độ Real Mode phải không, nhưng đừng lo, nó dễ thôi. Ví dụ, `LIMIT 15:0` có nghĩa là đoạn bit 0-15 của Descriptor chứa giá trị cho `limit`. Phần còn lại của nó là `LIMIT 19:16`. Vì thế, kích thước của giá trị  Limit là đoạn bit 0-19 hay là 20-bits. Giờ chúng ta sẽ xem xét kĩ từng trường một:

1. Limit[20-bits] gồm 2 đoạn 0-15,16-19 bits. Nó định nghĩa giá trị `length_of_segment - 1`. Phụ thuộc vào giá trị của `G`(Granularity) bit.

  * Nếu `G` (bit 55) là 0 và segment limit là 0, kích thước của  segment là 1 Byte
  * Nếu `G` là 1 và segment limit là 0, kích thước của segment là 4096 Bytes
  * Nếu `G` là 0 và segment limit là 0xfffff, kích thước của segment là 1 Megabytes
  * Nếu `G` là 1 and segment limit là 0xfffff, kích thước của segment là 4 Gigabytes

  Vì vậy, nói ngắn gọn lại, nó sẽ như sau:
  * Nếu G là 0, Limit có thể được hiểu là 1 Byte và kích thước lớn nhất của segment có thể là 1 Megabyte.
  * Nếu G là 1, Limit có thể được hiểu là 4096 Bytes = 4 KBytes = 1 Page và kích thước của segment có thể là 4 Gigabytes. Thực sự thì khi G bằng 1, giá trị Limit được shift sang trái 12 bits. Vì thế, 20 bits + 12 bits = 32 bits and 2<sup>32</sup> = 4 Gigabytes.

2. Base[32-bits] ở các đoạn (0-15, 32-39 and 56-63 bits). Nó định nghĩa địa chỉ bắt đầu của segment.

3. Type/Attribute (40-47 bits) định nghĩa loại segment và kiểu truy cập đến nó. 
  * Cờ `S` ở bit 44 chỉ ra loại descriptor. Nếu `S` là 0, đó là loại system segment, nếu `S` là 1, đó là code hoặc data segment (Stack segments là data segments cái sẽ được đọc/ghi theo segement).
  
Để kiểm tra một segment là code hay là data segment chúng ta có thể kiểm tra giá trị Ex(bit 43) Attribute của nó, cái được đánh dấu 0 ở hình trên. Nếu nó là 0, thì đó là Data segment ngược lại thì đó là code segment.

Các loại segment :

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

 Như thấy ở trên (bit 43) là `0` có nghĩa là _data_ segment, và bằng `1` có nghĩa là _code_ segment. Ba bit tiếp theo là (40, 41, 42, 43) là `EWA`(*E*xpansion *W*ritable *A*ccessible) hoặc CRA(*C*onforming *R*eadable *A*ccessible).
  * Nếu E(bit 42) là 0, expand up other wise expand down. Đọc thêm tại [đây](http://www.sudleyplace.com/dpmione/expanddown.html).
  * Nếu W(bit 41)( dành cho Data Segments) là 1, chỉ trường hợp này quyền ghi mới được phép, còn lại thì không. Chú ý rằng, quyền đọc thì luôn cho phép trên data segment.
  * A(bit 40) - segment này sẽ được access bởi processor hay không?.
  * C(bit 43) là bit conforming bit(được sử dụng khi chọn code). Nếu C là 1, cái segment code có thể được chạy từ quyền truy cập thấp hơn (lower level privilege) ví dụ là level user. Nếu C là 0, nó chỉ được phép chạy từ user có cùng mức truy cập (privilege level).
  * R(bit 41)(dành cho code segments). Nếu là 1, thì sẽ cho phép đọc ngược lại thì không. Quyền ghi không bao giờ được cho phép.

4. DPL[2-bits] (Descriptor Privilege Level) là các bit 45-46. Nó định nghĩa mức độ truy cập (privilege level) của segment. Nó có khoảng giá trị 0-3, trong đó 0 tương đương với quyền truy cập cao nhất (most privileged).

5. P flag(bit 47) - cho biết segment có được biểu diễn trên bộ nhớ hay không. Nếu P là 0, segment được hiểu là _invalid_ và process sẽ không đọc segment này.

6. AVL flag(bit 52) - Bit dự trữ. Bị bỏ qua trên Linux.

7. L flag(bit 53) - chỉ ra code segment có chứa native 64-bit code hay không. Bằng 1 có nghĩa là segment được chạy trong chế độ 64-bit.

8. D/B flag(bit 54) - Cờ Default/Big biểu diễn kích thước toán tử ví dụ 16/32 bits. Nếu nó được set thì được hiểu là 32 bit, còn lại là 16.

Segment registers chứa các segment selectors khi ở real mode. Tuy nhiên, ở protected mode, segment selector được handled theo cách khác. Mỗi Segment Descriptor có một Segment Selector, ở dạng một cấu trúc 16-bit structure:

```
15              3  2   1  0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

 trong đó,
* **Index** chỉ ra số lượng của descriptor trong GDT.
* **TI**(Table Indicator)  cho biết nên tìm kiếm các descriptor ở đâu. Nếu nó là 0 việc tìm kiếm được thực hiện trong Global Descriptor Table(GDT), ngược lại sẽ được tìm kiếm trong Local Descriptor Table(LDT).
* Và **RPL** là Requester's Privilege Level.

 Mọi segment register đều có phần nổi và chìm (visible and hidden part).
* Visible - Segment Selector sẽ được lưu ở đây 
* Hidden - Segment Descriptor(base, limit, attributes, flags)
 
 Các bước sau cần được thực hiện để tìm ra địa chỉ vật lý (physical address) ở chế độ protected mode:

* Cái segment selector phải được load vào một trong những segment registers
* CPU cố gắng tìm một segment descriptor bằng GDT address + Index từ selector và load descriptor vào phần *hidden*  của segment register
* Base address (from segment descriptor) + offset sẽ là địa chỉ tuyển tính (linear address) của segment hay chính là địa chỉ vật lý (physical address) (nếu phân trang được tắt đi).

 Về mặt mô tả hình học thì nó sẽ như sau:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

 Thuật toán để chuyển từ real mode sang protected mode có dạng như sau:

* Tắt ngắt (Disable interrupts)
* Mô tả và load GDT bằng câu lệnh lgdt (Describe and load GDT with `lgdt` instruction)
* Bất Protection bằng cách Set bit PE (Protection Enable) trong thanh ghi CR0 (Control Register 0)
* Nhảy đến code chạy trong chế độ protected mode

 Chúng ta sẽ xem xét quá việc chuyển đổi đầy đủ sang protected mode trong nhân Linux trong phần tiếp theo, nhưng trước khi đến đó, chúng ta sẽ làm một số chuẩn bị.

Cùng nhìn source của file [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). Chúng ta có thấy một vài đoạn mã thực hiện việc khởi tạo bàn phím, khởi tạo heap, etc... Hãy nhìn qua chúng một chút.

 Copy các tham số boot vào trang 0 ("zeropage")
--------------------------------------------------------------------------------

 Chúng ta bắt đầu từ hàm `main` quen thuộc trong file "main.c". Hàm đầu tiên được gọi trong hàm `main` là [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30). Nó thực hiện việc copy đoạn đầu của thiết lập nhân (kernel setup header) vào các trường của cấu trúc `boot_params`, cấu trúc này được định nghĩa ở [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113).

Cấu trúc `boot_params` chứa trường `struct setup_header hdr`. Cấu trúc này chứa những trường cùng tên được định nghĩa trong [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) và giá trị các trường sẽ được gán bởi boot loader lẫn thời điểm build kernel (kernel compile/build time). Hàm `copy_boot_params` thực hiện 2 thứ:

1. Copies giá trị `hdr` từ [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281) vào cấu trúc `boot_params`, hay chính xác là trường `setup_header`

2. Điều chỉnh con trỏ trỏ đến vị trí chứa dòng lệnh khởi nhân nếu nhân được load bằng giao thức dòng lệnh cũ.

Chú ý rằng, nó copy `hdr` bằng hàm `memcpy` được định nghĩa trong file [copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S). Hãy xem bên trong nó chút:

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

 Ồ, chúng ta vừa ở code C, đến đây lại quay lại assembly rồi :) Đầu tiên nhất chúng ta có thể thấy ở hàm `memcpy` này hay cả các hàm khác được định nghĩa tương tự, nó được mở đầu và kết thúc bởi 2 macros: `GLOBAL` và `ENDPROC`. `GLOBAL` được miêu tả trong file [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) cái sẽ định nghĩa chỉ thị dịch `globl` cùng với nhãn tương ứng cho nó. `ENDPROC` được mô tả tại file [include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h)  nó sẽ đánh dấu kí hiệu truyền vào `name` là một tên hàm function name và kết thúc bằng kích thước của kí hiệu `name` đó.

 Xử lý của `memcpy` là dễ hiểu. Đầu tiên, nó đẩy giá trị từ các thanh ghi `si` và `di` registers vào stack để giữ giá trị của chúng vì chúng sẽ được thay đổi sau đó `memcpy`. `memcpy` ( và các hàm khác trong copy.S) sử dụng cú pháp gọi `fastcall`. Vì thế, nó sẽ nhận các tham số đầu vào từ các thanh ghi `ax`, `dx` và `cx`. Lời gọi hàm sẽ `memcpy` sẽ trông giống như sau:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

 Vì thế,
* `ax` chứa địa chỉ của `boot_params.hdr`
* `dx` chứa địa chỉ của `hdr`
* `cx` sẽ chứa kích thước của `hdr` tính bằng bytes.

`memcpy` sẽ đặt địa chỉ của `boot_params.hdr` vào thanh ghi `di` và lưu kích thước của nó lên stack. Sau bước này nó sẽ dịch sang trái 2 bit (hay chia cho 4) và copy từ `si` vào `di` đúng 4 bytes. Sau bước này chúng ta sẽ phục hồi kích thước của `hdr`, bỏ đi 4 bytes và copy phần còn lại của dãy bytes từ `si` đến `di` theo từng byte 1 (hoặc có thể hơn). Khi đã phục hồi được giá trị `si` và `di` thì quá trình copy sẽ kết thúc.

Khởi tạo console 
--------------------------------------------------------------------------------

 Sau khi `hdr` được copy vào `boot_params.hdr`, bước tiếp theo là khởi tạo console bằng cách gọi hàm `console_init` được định nghĩa tại file [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c).

 Nó sẽ tìm xem có option `earlyprintk` trong câu lệnh khở động nhân không, và nếu tìm thấy, nó sẽ đọc địa chỉ cổng, baud rate của cổng serial rồi khởi động cổng serial được truyền cho đó. Giá trị của otpion `earlyprintk` trong dòng lệnh khởi động nhân có thể là một trong những giá trị sau:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

Sau quá trình khởi tạo console, chúng ta có thể thấy được output đầu tiên thông qua đoạn mã sau:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

Định nghĩa của hàm `puts` dùng ở trên nằm trong file [tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c). Nếu xem source của nó bạn có thể thấy đó là một vòng lặp kết hợp với hàm `putchar`. Nào cùng xem xét hàm `putchar` xem nó thực hiện cái gì:

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

`__attribute__((section(".inittext")))` có nghĩa là đoạn này được đặt trong section `.inittext`. Chúng ta có thể tìm thấy tên của nó trong file linker [setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19).

Trong thực thi của nó, đầu tiên nhất, hàm `putchar` kiểm tra xem có phải kí tự `\n` hay không, và nếu đúng thật, nó sẽ in ra `\r`. Sau đó nó đẩy kí tự ra màn hình VGA thông qua hàm của BIOS với lời gọi ngắt là `0x10`:

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

 Hàm `initregs` ở đây khởi tạo cấu trúc `biosregs` structure và gán ZERO cho biến cấu trúc `biosregs` đó sử dụng hàm khá quen thuộc trong C `memset`, sau đó nó điền các giá trị của thanh ghi tương ứng.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Cùng xem thực thi của hàm [memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36) này:

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

 Như bạn đã đọc được ở trên, nó sử dụng cú pháp gọi `fastcall` giống như cách hàm `memcpy` đã thực hiện, có nghĩa là nó sẽ lấy các tham số từ 3 thanh ghi `ax`, `dx` và `cx` .

Nói chung `memset` giống như các thực hiện hàm memset. Nó lưu giá trị thanh ghi `di` vào stack stack đưa giá trị`ax` vài `di` hay chính là chứa địa chỉ của cấu trúc `biosregs`. Tiếp theo đó là câu lệnh `movzbl`, nó sẽ copy giá trị `dl` vào 2 byte cuối của thanh ghi `eax`. 2 byte cao của thanh ghi `eax` này sẽ được điền bằng 0 hết.

Câu lệnh tiếp theo `imull` sẽ nhân `eax` với giá trị `0x01010101`. Đoạn này 4 byte cần thiết bởi vì `memset` sẽ copy 4 bytes đồng thời, nhân với 0x01 mỗi byte để lấy giá trị mà muốn fill. Ví dụ, chúng ta cần điền một cấu trúc với toàn giá trị `0x7` bằng memset chẳng hạn. `eax` sẽ chứa giá trị cần để sử dụng là `0x00000007`. Vì thế chúng ta đã nhân `eax` với `0x01010101`, và chúng ta được `0x07070707` và giờ copy 4 bytes này vào cấu trúc. `memset` sử dụng các lệnh `rep; stosl` để copy từ `eax` sang `es:di`.

Đoạn còn lại của hàm `memset` gần như giống với `memcpy`.

Sau khi cấu trúc `biosregs` được điền bằng `memset`, hàm `bios_putchar` sẽ gọi ngắt [0x10](http://www.ctyme.com/intr/rb-0106.htm) để thực hiện in ra kí tự tương ứng. tiếp theo của hàm putchar, nó sẽ kiểm tra cổng serial được khởi tạo hay chưa, để đẩy kí tự tương ứng ra với hàm [serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30) và thực hiện `inb/outb` nếu nó được set.

Khởi tạo Heap
--------------------------------------------------------------------------------

 Sau khi stack và bss section được chuẩn bị trong [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) (xem chương trước tại [part](linux-bootstrap-1.md)), nhân cũng cần khởi tạo [heap](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) nữa bằng hàm [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116).

Đầu tiên nhất `init_heap` kiểm tra cờ [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21) từ giá trị [`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) trong header thiết lập nhân và tính toán vị trí cuối của stack nếu cờ này được set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

hay nói cách khác `stack_end = esp - STACK_SIZE`.

Sau đó, vị trí cuối của heap, `heap_end` được tính như sau:
```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```
điều này có nghĩa là bằng `heap_end_ptr` hay `_end` + `512`(`0x200h`). Đoạn cuối sẽ kiểm tra xem `heap_end` có lớn hơn giá trị `stack_end` hay không. Nếu có thì`stack_end` được gán bằng `heap_end` để làm chúng bằng nhau.

Giờ thì heap đã được khởi tạo, chúng ta có thể sử dụng hàm `GET_HEAP`. Chúng ta sẽ thấy chúng được sử dụng như thế nào, và làm thế nào được sử dụng cũng như chúng được viết ra sao.

Xác nhận CPU (CPU validation)
--------------------------------------------------------------------------------

Chương tới chúng ta sẽ thấy đó là xác nhận CPU bằng hàm `validate_cpu` từ file source [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c).

Nó sẽ gọi hàm [`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102) truyền vào 2 giá trị cpu level hiện tại và cpu level yêu cầu tối thiểu sau đó kiểm tra xem kernel đang chạy ở đúng level hay không.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
`check_cpu` kiểm tra các cờ cpu's flags, có hay không có [long mode](http://en.wikipedia.org/wiki/Long_mode) trong trường hợp đó là CPU x86_64(64-bit), kiểm tra CPU Vendor và thực hiện các chuẩn bị tương ứng cho Vendor vendors như là tắt SSE+SSE2 trong trường hợp AMD nếu nó không có chẳng hạn, etc.

Phát hiện bộ nhớ
--------------------------------------------------------------------------------

Bước tiếp theo là phát hiện bộ nhớ bằng hàm `detect_memory`. `detect_memory` về cơ bản cung cấp một sơ đồ của các bộ nhớ RAM mà cpu có thể sử dụng. Nó sử dụng nhiều giao diện lập trình khác nhau để phát hiện bộ nhớ như `0xe820`, `0xe801` và `0x88`. Ở đây, chúng ta chỉ xem xét trường hợp **0xE820** thôi.

Cùng nhau nhìn hàm `detect_memory_e820` được viết như thế nào từ file source chứa nó tại [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c). Đầu tiên nhất, hàm `detect_memory_e820` khởi tạo một biến của cấu trúc `biosregs` như ta đã thấy ở trên, rồi điền vào các giá trị đặc biệt dành cho lời gọi kiểu `0xe820` này:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` chứa chỉ số của hàm ( trong trường hợp của chúng ta, đó là 0xe820)
* `cx` chứa kích thước của buffer, nơi mà sẽ chứa dữ thông tin về bộ nhớ 
* `edx` phải là giá trị ma thuật (magic number) có tên `SMAP`
* `es:di` phải chứa địa của buffer, nơi sẽ chứa thông tin về bộ nhớ 
* `ebx` phải là 0.

Sau đó là một vòng lặp để lấy thông tin về bộ nhớ. Nó bắt đầu bằng lời gọi ngắt `0x15` dành cho BIOS, mỗi lần gọi nó sẽ lấy ra một dòng từ bảng cấp phát địa chỉ (address allocation table). Để đọc dòng tiếp theo, chúng ta cần gọi ngắt này một lần nữa (chính vì thế mới có một vòng lặp). Trước lần gọi tiếp theo thanh ghi `ebx` phải chứa giá trị được trả về lúc trước:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Cuối cùng thì, nó sẽ lần lượt liệt kê tất cả thông tin từ bảng cấp phát địa chỉ (address allocation table), và ghi ghi dữ liệu có đang vào mang có tên `e820entry`, các thông tin này bao gồm:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

Bạn có thể thấy kết quả kiểu như sau từ dòng lệnh `dmesg`, trông kiểu như thế này:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Khởi tạo bàn phím (Keyboard initialization)
--------------------------------------------------------------------------------

 Tiếp đây là đến quá trình khởi tạo của bàn phím (keyboard) thông qua lời gọi hàm [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) tương ứng. Đầu tiên nhất, `keyboard_init` cũng khởi tạo các thanh ghi sử dụng hàm `initregs`, rồi gọi ngắt [0x16](http://www.ctyme.com/intr/rb-1756.htm) để lấy trạng thái của bàn phím.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
Sau bước này, nó sẽ gọi lại ngắt [0x16](http://www.ctyme.com/intr/rb-1757.htm) một lần nữa để thiết lập repeat rate và delay.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Lấy cách thông số khác (Querying)
--------------------------------------------------------------------------------

Nhóm các bước tiếp theo là truy vấn các tham số khác. Chúng ta sẽ không đi sau vào chi tiết các tham số này, mà để nó ở những phần sau. Ở đây, ta chỉ nhìn xem qua cá hàm truy vấn này:

Lời gọi [query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) gọi ngắt BIOD [0x15](http://www.ctyme.com/intr/rb-1594.htm) để lấy thông tin về số machine model, số sub-model, BIOS revision level, và các thuộc tính dành riêng cho phần cứng khác nữa:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

 Nó điền thanh ghi `ah` bằng giá trị `0xc0`, rồi gọi ngắt BIOS `0x15`. Sau khi ngắt này được chạy xong, nó sẽ kiểm tra cờ [carry flag](http://en.wikipedia.org/wiki/Carry_flag), nếu cờ này được set bằng 1, tức là BIOS không hỗ trợ [**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture). Nếu cơ này bằng 0, `ES:BX` sẽ chứa địa chỉ trỏ đến bảng thông tin hệ thống (system information table), cái mà nếu có sẽ kiểu như sau:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

 Sau đó, chúng ta gọi hàm `set_fs` truyền giá trị ở thanh ghi `es` cho nó. Thực thi của hàm `set_fs` này khá đơn giản:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

 Hàm này chứa một đoạn assembly inline, có nhiệm vụ lấy giá trị tham số `seg` và đẩy nó vào thanh ghi `fs`. Có rất nhiều hàm trong [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) giống như hàm `set_fs` này, ví dụ `set_gs`, `fs`, `gs` để đọc giá trị trong đó, vân vân...

Ở cuối hàm `query_mca`, nó đơn giản là copy cái bảng được chỉ bởi `es:bx` vào `boot_params.sys_desc_table` thôi.

Bước tiếp theo là lấy thông tin về [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) bằng lời gọi hàm `query_ist`. Đầu tiên nhất, nó sẽ kiểm tra CPU level và nếu đúng, nó gọi ngắt `0x15` để lấy thông tin và lưu kết quả vào `boot_params`.

Hàm [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) sausẽ lấy thông tin [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) từ BIOS. Hàm `query_apm_bios` này cũng gọi đến ngắt `0x15` trong BIOS, nhưng với giá trị  `ah` = `0x53` để kiểm tra cài đặt `APM`. Sau khi thực hiện xong ngắt `0x15`, hàm `query_apm_bios` kiểm tra chữ kí `PM` signature (nó phải bằng `0x504d`), rồi giá trị carry flag ( phải bằng 0 nếu `APM` được hỗ trợ) và giá trị của thanh ghi `cx`(nếu nó là 0x02, giao diện cho protected mode được hỗ trợ).

Sau đó, nó gọi ngắt `0x15` một lần nữa, nhưng với giá trị `ax = 0x5304` để ngắt kết nối giao diện `APM`, và chuyển sang giao diện ở chế độ 32-bit protected mode. Cuối cùng, nó điền giá trị cho biến `boot_params.apm_bios_info` từ các giá trị nó lấy được từ BIOS.

Chú ý là, hàm `query_apm_bios` chỉ được thực hiện nếu các giá trị cấu hình `CONFIG_APM` hoặc `CONFIG_APM_MODULE` được định nghĩa trong file cấu hình (của kernel):

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

Cuối cùng hàm [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122), sẽ lấy thông tin `Enhanced Disk Drive` (thông tin thêm về Drive) từ BIOS. Chúng ta sẽ cùng xem hàm này `query_edd` một chút.

Đầu tiên nhất, nó đọc giá trị option [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) từ tham số dòng lệnh kernel, nếu giá trị này được set là `off` thì hàm `query_edd` trả về luôn.

Nếu EDD được bật, `query_edd` đi qua các ổ cứng được BIOS hỗ trợ và lấy thông tin EDD bằng vòng lặp sau:

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

Giá trị vị trí `0x80` là chỗ của ổ cứng đầu tiên và giá trị của `EDD_MBR_SIG_MAX` là 16. Vòng lặp nãy sẽ tổng hợp các thông tin cho vào mảng của cấu trúc [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172). Hàm `get_edd_info` sẽ kiểm tra xem EDD có tồn tại hay không bằng ngắt `0x13` với tham số `ah` được gán giá trị `0x41` nếu EDD tồn tại, hàm `get_edd_info` lại gọi ngắt `0x13` một lần nữa, nhưng với tham số `ah` được gán giá trị `0x48` và `si` chứa địa chỉ của buffer mà EDD được lưu.

Phần kết
--------------------------------------------------------------------------------

 Đây là đoạn kết của phần 2 về những thứ bên trong Linux kernel. Ở phần tiếp theo, chúng ta sẽ xem xét các thiết lập video mode và những chuẩn bị còn lại trước khi chuyển sang protected mode và quá trình chuyển đổi trực tiếp sang nó.

Nếu bạn có bất cứ câu hỏi nào, hãy để lại comment hoặc ping cho tôi trên [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/master/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)

