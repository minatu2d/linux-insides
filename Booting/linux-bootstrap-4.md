Quá trình boot nhân. phần 4 - Kernel booting process. Part 4.
================================================================================

 Chuyển sang chế độ 64-bit mode
--------------------------------------------------------------------------------

 Đây là phần thứ 4 trong loại bài về `Kernel booting process` (quá trình boot nhân), nơi chúng ta đã thấy những bước đầu tiên khi chuyển sang [protected mode](http://en.wikipedia.org/wiki/Protected_mode), như kiểm tra xem cpu có hỗ trợ [long mode](http://en.wikipedia.org/wiki/Long_mode) hay không và [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions), rồi thì [paging](http://en.wikipedia.org/wiki/Paging), khởi tạo bảng page (page tables), rồi ở cuối ta đã nói đến việc chuyển sang [long mode](https://en.wikipedia.org/wiki/Long_mode).

**NOTE: there will be much assembly code in this part, so if you are not familiar with that, you might want to consult a book about it**

Ở [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md) trước, chúng ta đang dừng lại ở đoạn nhảy đến điểm đầu vào 32-bit - 32-bit entry point trong file [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S):

```assembly
jmpl	*%eax
```

Chúng ta lại thấy thanh ghi `eax` chứa địa chỉ của điểm đầu vào 32-bit. Chúng ta có thể đọc thêm về thanh ghi này tại [linux kernel x86 boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt):

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
```
Ý là:
```
Khi sử sụng bzImage, vị trị kernel trong protected mode sẽ nằm ở địa chỉ 0x100000
```


Nào, cùng nhau kiểm tra xem đó nó có đúng không bằng cách xem 32-bit entry point được gán như thế nào:

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

Bạn có thể thấy ở đây thanh ghi `cs` chứa giá trị `0x10` (bạn có thể nhớ từ phần trước, đây là index thứ 2 trong bảng GDT -  Global Descriptor Table), thanh ghi `eip` chứa `0x100000` và địa chỉ cơ sở (base address) của tất cả các segment gồm cả code segment đều là zero. Vì thế ta có thể tính ra địa chỉ vật lý của nó, nó sẽ là `0:0x100000` hay đơn giản là `0x100000`, như được chỉ ra trong giảo thức boot nhân. Bây giờ, chúng ta sẽ đi từ điểm đầu vào 32-bit - 32-bit entry point.

 Điểm đầu vào 32-bit - 32-bit entry point
--------------------------------------------------------------------------------

Bạn có thể tìm thấy định nghĩa của điểm đầu vào này tại file asm [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

Hãy để ý một chút, file chứa định nghĩa nằm trong thư mục có tên  `compressed`, tại sao lại như thế? Đó là vì, `bzimage` là file nén chứa `vmlinux(binary của nhân) + header + kernel setup code(code thiết lập nhân)`. Chúng ta đã xem xét phần code thiết lập nhân - kernel setup code trong các phần trước rồi. Vì thế, mục tiêu của `head_64.S` thực hiện các chuẩn bị để chuyển sang long mode, chuyển vào đó xong sẽ giải nén nhân. Chúng ta sẽ thấy tất cả các bước cho đến phần giải nén nhân - kernel decompression trong phần này.

Có 2 file trong thư mục `arch/x86/boot/compressed`:

* [head_32.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)

nhưng chúng ta chỉ xem file `head_64.S` thôi vì, có thể bạn còn nhớ, cuốn sách này chỉ nói về `x86_64` thôi; `head_32.S` không được nói đến trong trường hợp của chúng ta. Hãy xem file [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile). Ở đó, bạn có thể thấy một rule để build như sau:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

 Ở đây, chú ý rằng có file `$(obj)/head_$(BITS).o`. Tức là, cho phép chọn file nào được link thông qua việc gán giá trị biến cho biến `$(BITS)`, hoặc là head_32.o hoặc là head_64.o.   `$(BITS)` được định nghĩa ở [arch/x86/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/Makefile) dựa trên nội dung file .config:

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

 Giờ, chúng ta đã biết chút ý về điểm bắt đầu, nào hãy xem nó.

 Reload các segment nếu cần - Reload the segments if needed
--------------------------------------------------------------------------------

Như đã nhắc từ trên, chúng ta sẽ bắt đầu từ file asm [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S). Đầu tiên, chúng ta sẽ xem một thuộc tính segment đặc biệt - special section attribute trước định nghĩa `startup_32` trong đoạn dưới đây:

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

Cái `__HEAD` này là một macro được định nghĩa trong file header [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) và được mở rộng thành định nghĩa như sau:

```C
#define __HEAD		.section	".head.text","ax"
```

với tên `.head.text` và cờ `ax`. Trong trường hợp của chúng ta, cờ này sẽ chỉ cho chúng ta biết cái section này là [executable](https://en.wikipedia.org/wiki/Executable) tức là chạy được hay nói cách khác là chứa code. Chúng ta có thể tìm thấy định nghĩa section này trong script dành cho linker [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S):

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
```

Nếu bạn không quen với ngôn ngữ - `GNU LD` linker scripting, bạn có thể xem thêm thông tin trong [documentation](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts). Nói ngắn gọn, kí hiệu `.` (dấu chấm) là một biến đặc biệ của linker - đếm vị trí (location counter). Giá trị được gán đến nó là chỉ vị trí tương đối đến offset trong segement đó. Trong trường hợp của chúng ta, ban đầu gán 0 cho biến đếm vị trí. Điều này có nghĩa rằng, code của chúng ta vẫn được link để chạy từ vị trí có offset `0` trên bộ nhớ. Thêm nữa, bạn có thể thấy thêm thông tin trong đoạn comment sau:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```
Ý là:
```
Chú ý là, các phần code của head_64.s giả định rằng startup_32 nằm ở địa chỉ 0
```

Ok, giờ chúng ta biết là chúng ta đang ở đâu, giờ là luc thích hợp để xem bên trong hàm `startup_32`.

Trong đoạn bắt đầu của hàm `startup_32`, chúng ta thấy lệnh `cld` được gọi để thực hiện xóa bit `DF` trong thanh ghi cờ - [flags](https://en.wikipedia.org/wiki/FLAGS_register). Khi các cờ này được xóa, thì tất cả các thao tác với chuỗi như [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) và các thao tác khác sẽ tăng index bằng thanh ghi `esi` hoặc `edi`. Chúng ta cần xóa các cờ hướng này vì sau đó ta sẽ dùng các thao tác chuỗi để xóa vùng trống dành cho bảng page - page tables, etc.

Sau khi xóa bit `DF`, tiếp đến là kiểm tra cờ `KEEP_SEGMENTS` từ trường `loadflags` trong header setup nhân. Nếu bạn nhớ chúng ta đã từng thấy `loadflags` trong các [part](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html) đầu của cuốn sách này. Ở đó, chúng ta đã kiểm tra cờ `CAN_USE_HEAP` xem có thể sử dụng heap hay không. Giờ, ở đây chúng ta kiểm tra cờ `KEEP_SEGMENTS`. Cờ này được miêu tả trong tài liệu [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt):

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
	a base of 0 (or the equivalent for their environment).
```
Ý là:
```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - Nếu là 0, load lại các thanh ghi segment trong điểm đầu vào 32bit.
  - Nếu là 1, đừng load lại segment registers trong điều đầu vào 32bit.
    Giả định rằng %cs %ds %ss %es được dùng để set flat segments (segement không bị phân mảnh) với base bằng 0 
    (hoặc tương đương như thế trong môi trường của chúng).
```

Vì thế, nếu bit `KEEP_SEGMENTS` không được bật trong cờ `loadflags`, chúng ta cần reset các thanh ghi segment `ds`, `ss` và `es` thành các segment không bị phân mảnh với `0`. Cái chúng ta sẽ làm là:

```C
	testb $(1 << 6), BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

Hãy nhớ rằng `__BOOT_DS` bằng `0x18` ( index hay vị trí của segment data trong [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)). Nếu giá trị `KEEP_SEGMENTS` được set, chúng ta sẽ nhảy đến vị trí có nhãn `1f` hoặc update các thanh ghi segment theo `__BOOT_DS` nếu nó không được set. Nó khá dễ hiểu nhưng, có một điểm thú vị ở đây. Nếu bạn đọc phần trước [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md), bạn có thể nhớ rằng, chúng ta đã update các giá trị thanh ghi segment một cách đúng đắn lúc chúng ta chuyển sang chế độ [protected mode](https://en.wikipedia.org/wiki/Protected_mode) trong file [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S). Vậy tại sao chúng ta cần quan tâm đến giá trị của mấy thanh ghi segment này làm gì nữa? Câu trở thật đơn giản. Nhân Linux có một giao thức boot ở 32-bit và nếu bootloader sử dụng chính giao thức đó để load hết code của nhân Linux trước hàm `startup_32` thì không ổn. Trong trường hợp này, hàm `startup_32` phải là điểm vào đầu tiên nhất của nhân Linux sau bootloader và không có gì để đảm bảo giá trị các thanh ghi segment còn được giữ nguyên trạng thái cũ.

Sau khi chúng ta kiểm tra giá trị cờ `KEEP_SEGMENTS` và đặt giá trị thích hợp vào thanh ghi segment, bước tiếp theo là tính toán sự sai khác giữa vị trí chúng ta đã load vào và vị trí được biên dịch để chạy. Nhớ rằng `setup.ld.S` có chứa định nghĩa sau: `. = 0` ở chỗ bắt đầu của section `.head.text`. Có nghĩa là code trong section này sẽ được biên dịch để chạy từ địa chỉ `0`. Bạn có thể thấy trong đầu ra của câu lệnh quen thuộc trong Linux `objdump`:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

Câu lệnh `objdump` cho chúng ta biết địa chỉ của hàm `startup_32` bằng `0`. Nhưng thực sự lại không phải vậy. Mục tiêu hiện tại là biết chỗ chúng ta thực sự nằm là ở chỗ nào. Để biết cái này thì khá đơn giản khi ở chế độ [long mode](https://en.wikipedia.org/wiki/Long_mode), bởi vì nó hỗ trợ đánh địa chỉ tương đối `rip` - `rip` relative addressing, nhưng giờ chúng ta đang ở trong [protected mode](https://en.wikipedia.org/wiki/Protected_mode) thôi. Chúng ta sẽ sử dụng một cách quen thuộc để biết địa chỉ của hàm `startup_32`. Chúng ta tạo một nhãn, thực hiện lời gọi đến nhãn này, ở đó lấy giá trị từ đỉnh stack (pop) đưa vào một thanh ghi:

```assembly
call label
label: pop %reg
```

Sau bước này, thanh ghĩ sẽ chứa địa chỉ của nhãn. Hãy cùng xem một đoạn code tương tự để xác định địa chỉ của hàm `startup_32` trong nhân Linux:

```assembly
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:  popl	%ebp
	subl	$1b, %ebp
```

Như đã biết từ phần trước, cái thanh ghi `esi` chứa địa chỉ của cấu trúc [boot_params](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113), cái chỗ mà sẽ được điền trước khi nhảy sang chế độ protected mode. Cấu cấu trúc `boot_params` chứa một trường đặc biệt tên là `scratch` với offset là `0x1e4`. Trường 4 byte đặc biệt này sẽ là stack tạm thời cho lệnh asm `call`. Chúng ta đang lấy địa chỉ của trường `scratch` + 4 bytes và đặt nó vào thanh ghi `esp`. Chúng ta đã cộng `4` bytes vào địa chỉ của trường `BP_scratch` bởi vì đơn giản như đã miêu tả, nó sẽ là một stack, và nó sẽ phình ra theo chiều từ trên xuống dưới trong kiến trúc `x86_64`. Vì thế con trỏ của chúng ta sẽ chỉ đến đỉnh của stack. Sau đó chúng ta có thể thấy cái xử lý tương tự như tôi đã miêu tả ở trên. Chúng ta thực hiện một lời gọi đến nhãn `1f`, rồi đặt địa chỉ của nhãn này vào thanh ghi `ebp`, bởi vì địa chỉ của nhãn được đưa vào đỉnh stack khi thực hiện lệnh asm `call` rồi. Vì thế, giờ chúng ta đã có địa chỉ của nhãn `1f` rồi và rất dễ dàng để lấy được địa chỉ của hàm `startup_32`. Chúng ta cần trừ địa chỉ hiện tại một lượng bằng đúng địa chỉ lấy được từ đỉnh stack:

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - real physical address
                     |                       |
                     |                       |
                     +-----------------------+
```

Hàm `startup_32` được linked để chạy từ địa chỉ `0x0` điều này có nghĩa rằng `1f` sẽ có địa chỉ bằng `0x0 + offset đến nhãn 1f`, tương đương với `0x21` bytes. Cái thanh ghi `ebp` chứa địa chỉ vật lý thực - real physical address của nhãn `1f`. Vì thế, nếu chúng ta trừ một lượng `1f` từ `ebp` chúng ta sẽ có được địa chỉ vật lý của `startup_32`. Giao thức boot nhân Linux [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) địa chỉ base của kernel trong chế độ protected mode là `0x100000`. Chúng ta có thể kiểm tra điều này bằng [gdb](https://en.wikipedia.org/wiki/GNU_Debugger). Bật debugger và đặt breakpoint ở địa chỉ `1f` address, tức địa chỉ thực là `0x100021`. Nếu là đúng, chúng ta sẽ thấy giá trị `0x100021` trong thanh ghi `ebp`:

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

Nếu chúng ta thực hiện lệnh asm tiếp theo, tức là `subl $1b, %ebp`, chúng ta sẽ thấy:

```
nexti
...
ebp            0x100000	0x100000
...
```

OK, vậy là đúng rồi. Địa chỉ của hàm `startup_32` nằm ở `0x100000`. Sau khi biết địa chỉ của nhãn `startup_32`, chúng ta có thể chuẩn bị để chuyển sang chế độ [long mode](https://en.wikipedia.org/wiki/Long_mode). Mục tiêu tiếp theo của chúng ta là thiết lập stack kiểm tra để chằng rằng CPU hỗ trợ chế độ long mode [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions).

Thiết lập stack và kiểm tra CPU
--------------------------------------------------------------------------------

Chúng ta không thể thiết lập stack trong khi không biết địa chỉ thực của nhãn `startup_32`. Chúng ta có thể tưởng tượng rằng stack giống như một cái mảng và thanh ghi chứa con trỏ stack `esp` sẽ trỏ đến phần tử cuối cùng của mảng đó. Tất nhiên chúng ta có thể định nghĩa một mảng trong code, nhưng bạn cần biết địa chỉ thực để cấu hình con trỏ stack cho đúng. Cùng nhìn vào code xem sao:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

Cái nhãn `boot_stack_end`, cũng được định nghĩa trong file mã nguồn asm [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) sẽ được đưa vào section [.bss](https://en.wikipedia.org/wiki/.bss):

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

Đầu tiên nhất, đặc địa chỉ của nhãn `boot_stack_end` vào thanh ghi `eax` register, vì thế thanh ghi `eax` sẽ chứa địa chỉ của `boot_stack_end` nơi nó được linked, hay nằm ở vị trí tương đối `0x0 + boot_stack_end`. Để lấy địa chỉ thực của `boot_stack_end`, chỉ cần cộng vị trí tương đối với địa chỉ thực của `startup_32` là xong. Bạn có thể nhớ, chúng ta đã tìm ra địa chỉ này ở trên và để vào thanh ghi `ebp` rồi. Cuối cùng, thanh ghi `eax` sẽ chứa địa chỉ thực của `boot_stack_end` và giờ chúng ta cần đặt nó con trỏ stack.

Sau khi chúng thiết lập stack, tiếp theo sẽ là xác nhận CPU. Khi chúng ta định thực hiện việc chuyển sang `long mode`, chúng ta cần kiểm tra xem CPU có hỗ trợ `long mode` và `SSE` không. Điều này được thực hiện bằng cách gọi hàm `verify_cpu`:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

Hàm này được định nghĩa trong trong file asm [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/verify_cpu.S) và đơn giản là chứa một loạt lời gọi đến lệnh asm [cpuid](https://en.wikipedia.org/wiki/CPUID). Lời gọi này được sử dụng để lấy thông tin từ CPU. Trong trường hợp của chúng ta, nó sẽ kiểm tra  `long mode` và `SSE` xem có được hỗ trợ không, rồi trả về `0` nếu có hoặc `1` nếu không được hỗ trợ thanh ghi `eax`.

Nếu giá trị của `eax` khác zero, chúng ta sẽ nhảy sang nhãn `no_longmode`, ở đó nó đơn giản là dừng CPU bằng cách gọi hàm `hlt` và không có ngắt phần cứng nào xảy ra hết:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

Nếu giá trị thanh ghi `eax` bằng zero, mọi thứ ok và chúng ta sẽ tiếp tục.

Tính toán địa chỉ để tái định vị
--------------------------------------------------------------------------------

Bước tiếp theo, là tính toán địa chỉ để định vị lại cho việc giải nén - decompress nếu cần cần giải nén nhân. Đầu tiên chúng ta cần viết với nhân thì `relocatable` nghĩa là gì. Chúng ta đã biết rằng, địa chỉ base của đầu vào 32-bit - 32bit entry point của Linux kernel là `0x100000`, nhưng đó chỉ là điểm đầu vào 32-bit. Địa chỉ base của nhân Linux được xác định bởi giá trị cấu hình nhân `CONFIG_PHYSICAL_START`. Giá trị mặc định của nó là `0x1000000` hoặc `16 MB`. Vấn đề chính ở đây là nếu nhân Linux kernel bị crashed, người phát triển nhân phải `cứu nhân` - `rescue kernel` để thực hiện dump bằng [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt) được cấu hình để load từ một địa chỉ khác. Nhân Linux cung cấp một lựa chọn cấu hình đặc biệt để giải quyết vấn đề này, đó là: `CONFIG_RELOCATABLE`. Như một đoạn sau đây ở trong tài liệu về nhân Linux:

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

Nói đơn giản, thì nhân Linux với cùng một cấu hình có thể boot từ nhiều địa chỉ khác nhau. Về mặt kĩ thuật, điều này có thể được thực hiện bằng cách biên dịch bộ giả nén nhân - decompressor ở dạng code không phụ thuộc vị trí [position independent code](https://en.wikipedia.org/wiki/Position-independent_code). Nếu nhìn vào [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile), chúng ta sẽ thấy đoạn dành cho decompressor được biên dịch với cờ `-fPIC`:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

Khi bạn sử dụng code phụ thuộc vị trí position-independent code an address is obtained by adding the address field of the command and the value of the program counter. We can load code which uses such addressing from any address. That's why we had to get the real physical address of `startup_32`. Now let's get back to the Linux kernel code. Our current goal is to calculate an address where we can relocate the kernel for decompression. Calculation of this address depends on `CONFIG_RELOCATABLE` kernel configuration option. Let's look at the code:

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
1:
	addl	$z_extract_offset, %ebx
```

Remember that the value of the `ebp` register is the physical address of the `startup_32` label. If the `CONFIG_RELOCATABLE` kernel configuration option is enabled during kernel configuration, we put this address in the `ebx` register, align it to a multiple of `2MB` and compare it with the `LOAD_PHYSICAL_ADDR` value. The `LOAD_PHYSICAL_ADDR` macro is defined in the [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/boot.h) header file and it looks like this:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

As we can see it just expands to the aligned `CONFIG_PHYSICAL_ALIGN` value which represents the physical address of where to load the kernel. After comparison of the `LOAD_PHYSICAL_ADDR` and value of the `ebx` register, we add the offset from the `startup_32` where to decompress the compressed kernel image. If the `CONFIG_RELOCATABLE` option is not enabled during kernel configuration, we just put the default address where to load kernel and add `z_extract_offset` to it.

After all of these calculations we will have `ebp` which contains the address where we loaded it and `ebx` set to the address of where kernel will be moved after decompression.

Preparation before entering long mode
--------------------------------------------------------------------------------

When we have the base address where we will relocate the compressed kernel image, we need to do one last step before we can transition to 64-bit mode. First we need to update the [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table):

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

Here we put the base address from `ebp` register with `gdt` offset into the `eax` register. Next we put this address into `ebp` register with offset `gdt+2` and load the `Global Descriptor Table` with the `lgdt` instruction. To understand the magic with `gdt` offsets we need to look at the definition of the `Global Descriptor Table`. We can find its definition in the same source code [file](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
	.data
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x0000000000000000	/* NULL descriptor */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

We can see that it is located in the `.data` section and contains five descriptors: `null` descriptor, kernel code segment, kernel data segment and two task descriptors. We already loaded the `Global Descriptor Table` in the previous [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md), and now we're doing almost the same here, but descriptors with `CS.L = 1` and `CS.D = 0` for execution in `64` bit mode. As we can see, the definition of the `gdt` starts from two bytes: `gdt_end - gdt` which represents last byte in the `gdt` table or table limit. The next four bytes contains base address of the `gdt`. Remember that the `Global Descriptor Table` is stored in the `48-bits GDTR` which consists of two parts:

* size(16-bit) of global descriptor table;
* address(32-bit) of the global descriptor table.

So, we put address of the `gdt` to the `eax` register and then we put it to the `.long	gdt` or `gdt+2` in our assembly code. From now we have formed structure for the `GDTR` register and can load the `Global Descriptor Table` with the `lgtd` instruction.

After we have loaded the `Global Descriptor Table`, we must enable [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) mode by putting the value of the `cr4` register into `eax`, setting 5 bit in it and loading it again into `cr4`:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Now we are almost finished with all preparations before we can move into 64-bit mode. The last step is to build page tables, but before that, here is some information about long mode.

Long mode
--------------------------------------------------------------------------------

[Long mode](https://en.wikipedia.org/wiki/Long_mode) is the native mode for [x86_64](https://en.wikipedia.org/wiki/X86-64) processors. First let's look at some differences between `x86_64` and the `x86`.

The `64-bit` mode provides features such as:

* New 8 general purpose registers from `r8` to `r15` + all general purpose registers are 64-bit now;
* 64-bit instruction pointer - `RIP`;
* New operating mode - Long mode;
* 64-Bit Addresses and Operands;
* RIP Relative Addressing (we will see an example of it in the next parts).

Long mode is an extension of legacy protected mode. It consists of two sub-modes:

* 64-bit mode;
* compatibility mode.

To switch into `64-bit` mode we need to do following things:

* Enable [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
* Build page tables and load the address of the top level page table into the `cr3` register;
* Enable `EFER.LME`;
* Enable paging.

We already enabled `PAE` by setting the `PAE` bit in the `cr4` control register. Our next goal is to build the structure for [paging](https://en.wikipedia.org/wiki/Paging). We will see this in next paragraph.

Early page table initialization
--------------------------------------------------------------------------------

So, we already know that before we can move into `64-bit` mode, we need to build page tables, so, let's look at the building of early `4G` boot page tables.

**NOTE: I will not describe the theory of virtual memory here. If you need to know more about it, see links at the end of this part.**

The Linux kernel uses `4-level` paging, and we generally build 6 page tables:

* One `PML4` or `Page Map Level 4` table with one entry;
* One `PDP` or `Page Directory Pointer` table with four entries;
* Four Page Directory tables with a total of `2048` entries.

Let's look at the implementation of this. First of all we clear the buffer for the page tables in memory. Every table is `4096` bytes, so we need clear `24` kilobyte buffer:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

We put the address of `pgtable` plus `ebx` (remember that `ebx` contains the address to relocate the kernel for decompression) in the `edi` register, clear the `eax` register and set the `ecx` register to `6144`. The `rep stosl` instruction will write the value of the `eax` to `edi`, increase value of the `edi` register by `4` and decrease the value of the `ecx` register by `1`. This operation will be repeated while the value of the `ecx` register is greater than zero. That's why we put `6144` in `ecx`.

`pgtable` is defined at the end of [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly file and is:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

As we can see, it is located in the `.pgtable` section and its size is `24` kilobytes.

After we have got buffer for the `pgtable` structure, we can start to build the top level page table - `PML4` - with:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Here again, we put the address of the `pgtable` relative to `ebx` or in other words relative to address of the `startup_32` to the `edi` register. Next we put this address with offset `0x1007` in the `eax` register. The `0x1007` is `4096` bytes which is the size of the `PML4` plus `7`. The `7` here represents flags of the `PML4` entry. In our case, these flags are `PRESENT+RW+USER`. In the end we just write first the address of the first `PDP` entry to the `PML4`.

In the next step we will build four `Page Directory` entries in the `Page Directory Pointer` table with the same `PRESENT+RW+USE` flags:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

We put the base address of the page directory pointer which is `4096` or `0x1000` offset from the `pgtable` table in `edi` and the address of the first page directory pointer entry in `eax` register. Put `4` in the `ecx` register, it will be a counter in the following loop and write the address of the first page directory pointer table entry to the `edi` register. After this `edi` will contain the address of the first page directory pointer entry with flags `0x7`. Next we just calculate the address of following page directory pointer entries where each entry is `8` bytes, and write their addresses to `eax`. The last step of building paging structure is the building of the `2048` page table entries with `2-MByte` pages:

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

Here we do almost the same as in the previous example, all entries will be with flags - `$0x00000183` - `PRESENT + WRITE + MBZ`. In the end we will have `2048` pages with `2-MByte` page or:

```python
>>> 2048 * 0x00200000
4294967296
```

`4G` page table. We just finished to build our early page table structure which maps `4` gigabytes of memory and now we can put the address of the high-level page table - `PML4` - in `cr3` control register:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

That's all. All preparation are finished and now we can see transition to the long mode.

Transition to the 64-bit mode
--------------------------------------------------------------------------------

First of all we need to set the `EFER.LME` flag in the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) to `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Here we put the `MSR_EFER` flag (which is defined in [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h#L7)) in the `ecx` register and call `rdmsr` instruction which reads the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) register. After `rdmsr` executes, we will have the resulting data in `edx:eax` which depends on the `ecx` value. We check the `EFER_LME` bit with the `btsl` instruction and write data from `eax` to the `MSR` register with the `wrmsr` instruction.

In the next step we push the address of the kernel segment code to the stack (we defined it in the GDT) and put the address of the `startup_64` routine in `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

After this we push this address to the stack and enable paging by setting `PG` and `PE` bits in the `cr0` register:

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

and execute:

```assembly
lret
```

instruction. Remember that we pushed the address of the `startup_64` function to the stack in the previous step, and after the `lret` instruction, the CPU extracts the address of it and jumps there.

After all of these steps we're finally in 64-bit mode:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

That's all!

Conclusion
--------------------------------------------------------------------------------

This is the end of the fourth part linux kernel booting process. If you have questions or suggestions, ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new).

In the next part we will see kernel decompression and many more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
