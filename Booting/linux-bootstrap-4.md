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

Khi bạn sử dụng code không phụ thuộc vị trí position-independent code, một địa chỉ sẽ được tính bằng cách cộng giá trị trường địa chỉ của câu lệnh và giá trị của program counter (PC). Chúng ta có thể load code kiểu như vậy bằng bất cứ địa chỉ nào. Đó lại tại sao tôi phải lấy địa chỉ vật lý thực của hàm `startup_32`. Nào, giờ chúng ta trở lại với phần code của nhân Linux. Mục tiêu của chúng ta là tính toán lại địa chỉ chỗ chúng ta đặt vị trí lại cho nhân khi giải nén. Cách tính địa chỉ này phụ thuộc vào giá trị cấu hình `CONFIG_RELOCATABLE`. Hãy xem phần code của nó:

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

Nhớ rằng, giá trị thanh ghi `ebp` chính là địa chỉ vật lý của nhãn `startup_32`. Nếu giá trị cấu hình nhân `CONFIG_RELOCATABLE` được bật, chúng ta sẽ đặt địa chỉ này vào thanh ghi `ebx`, dịch nó theo một số nguyên lần của `2MB`  và so sánh nó với giá trị địa chỉ `LOAD_PHYSICAL_ADDR`. Cái macro `LOAD_PHYSICAL_ADDR` được định nghĩa trong [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/boot.h) như sau:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

Như chúng ta thấy, nó đơn giản được mở rộng thành một phép tính toán trên giá trị `CONFIG_PHYSICAL_ALIGN`, chính là địa chỉ vật lý mà kernel được load vào. Sau khi so sánh `LOAD_PHYSICAL_ADDR` và giá trị thanh ghi `ebx`, chúng ta cộng thêm một offset tính từ `startup_32`, hay chính là nơi sẽ giải nén nhân. Nếu giá trị cấu hình nhân `CONFIG_RELOCATABLE` không được bật, chúng ta đơn giản là đặt địa chỉ được dùng để load nhân rồi cộng thêm `z_extract_offset` vào.

Sau tất cả các bước tính toán trên, chúng ta sẽ có trong thanh ghi `ebp` chứa địa chỉ nơi chúng ta đã load nhân và thanh ghi `ebx` chứa địa chỉ nhân sau khi nó được giải nén.

Các chuẩn bị trước khi vào long mode
--------------------------------------------------------------------------------

Khi chúng ta có địa chỉ nơi sử dụng để đặt vị trí của ảnh nén của nhân, chúng ta cần thực hiện bước cuối cùng trước khi chuyển sang chế độ 64-bit. Đầu tiên, chúng ta cần cập nhật [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table):

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

 Ở đây, chúng ta cần đặt địa chỉ base từ thanh ghi `ebp` và `gdt` offset vào thanh ghi `eax`. Tiếp theo, chúng ta sẽ thêm địa chỉ này vào thanh ghi `ebp` với offset bằng `gdt+2` và load `Global Descriptor Table` bằng lệnh `lgdt` . Để hiểu cái ma thuật với `gdt` offsets này chúng ta cần xem định nghĩa về `Global Descriptor Table`. Chúng ta có thể thấy định nghĩa của nó ở cùng file source code [file](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

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

Chúng ta có thể thấy rằng, nó được đặt ở `.data` section và chứa 5 descriptor: `null` descriptor, kernel code segment, kernel data segment và 2 task descriptors. Chúng ta đã load `Global Descriptor Table` vào từ [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md) trước  rồi, giờ làm gần như giống như thế ở đây, nhưng các descriptor sẽ có `CS.L = 1` and `CS.D = 0` để chạy trong `64` bit mode. Như chúng ta thấy, định nghĩa của `gdt` bắt đầu bằng 2 bytes: `gdt_end - gdt` cái biểu diễn byte cuối cùng của bảng `gdt` table hay giới hạn của table. Bốn bytes sau đó chứa địa chỉ `gdt`. Hãy nhớ rằng `Global Descriptor Table` được lưu dạng `48-bits GDTR` chứa 2 phần sau:

* size(16-bit) of global descriptor table;
* address(32-bit) of the global descriptor table.

Vì thế, chúng ta sẽ đặt địa chỉ của `gdt` vào thanh ghi `eax` và đặt chúng vào `.long gdt` hoặc `gdt+2` trong code asm của chúng ta. Từ thời điểm này, chúng ta đã cấu trúc cho thanh ghi `GDTR`, giờ có thể load `Global Descriptor Table` bằng lệnh asm `lgtd`.

Sau khi chúng ta load `Global Descriptor Table`, chúng ta phải bật [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) và đặt giá trị ở thanh ghi `cr4` vào thanh ghi `eax`, thiết lập 5 bit rồi load vào thanh ghi `cr4`:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Đến đây, chúng ta cũng gần như hoàn tất các bước chuẩn bị để vào 64-bit mode rồi. Bước cuối cùng là tạo các bảng trang - page tables, nhưng trước đó, có một vài thông tin về cái long mode.

Long mode
--------------------------------------------------------------------------------

[Long mode](https://en.wikipedia.org/wiki/Long_mode) là native mode của các bộ xử lý [x86_64](https://en.wikipedia.org/wiki/X86-64). Đầu tiên hãy xem sự khác nhau giữa `x86_64` và `x86`.

`64-bit` cung cấp các tính năng sau:

* 8 thanh ghi mới cho sử dụng chung `r8` đến `r15` được thêm vào khi ở 64-bit;
* Con trỏ lệnh 64-bit - 64-bit instruction pointer - `RIP`;
* Thêm chế độ hoạt động mới - Long mode;
* Địa chỉ và các toán hạng 64-bit;
* Đánh địa chỉ tương đối RIP - RIP Relative Addressing (chúng ta sẽ thấy một vài ví dụ ở phần sau).

Long mode là mở rộng của chế độ protected mode hiện có. Nó bao gồm 2 sub-modes:

* mode 64-bit -  64-bit mode;
* mode tương thích - compatibility mode.

Để chuyển sang mode `64-bit`, chúng ta cần thêm những thứ sau:

* Bật [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
* Tạo bảng trang (page tables) và load địa chỉ của trang đầu tiên vào thanh ghi `cr3`;
* Bật `EFER.LME`;
* Bật phân trang - paging.

Chúng ta đã bật `PAE` bắt các set bit `PAE` trong thanh ghi điều khiển `cr4` rồi. Việc tiếp theo là tạo cấu trúc phục vụ việc phân trang - [paging](https://en.wikipedia.org/wiki/Paging). Chúng ta sẽ thấy ở chương sau.

Khởi tạo bảng trang ban đầu - Early page table initialization
--------------------------------------------------------------------------------

Như vậy, chúng ta đã biết rằng, trước khi chuyển sang mode `64-bit`, chúng ta cần tạo bảng trang (page tables), vì thế, cùng nhau xem việc tạo bảng trang boot `4G` lúc đầu.

**NOTE: Tôi sẽ không nói lại lý thuyết về bộ nhớ ảo ở đây. Nếu bạn muốn biết thêm về nó, hãy xem các link ở cuối bài.**

Nhân Linux sử dụng phân trang `4-level`, và nói chung chúng ta sẽ tạo 6 bảng tables:

* Một bảng `PML4` hay `Page Map Level 4` với một entry;
* Một bảng `PDP` hay `Page Directory Pointer` với bốn entry;
* Bốn bảng thư mục trang - Page Directory với tổng cộng `2048` entry.

Cùng xem chúng được tạo ra trong code như thế nào. Đầu tiên, xóa buffer dành cho các bảng trang trong bộ nhớ. Mỗi bảng sẽ có `4096` bytes, vì thế chúng ta cần xóa một buffer kích thước `24` kilobyte:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

Chúng ta đặt giá trị tổng gồm địa chỉ của `pgtable` và `ebx` (nhớ rằng `ebx` chứa địa chỉ để đặt lại kernel sau khi giải nén) vào thanh ghi `edi`, xóa thanh ghi `eax` và gán giá trị thanh ghi `ecx` là `6144`. Lệnh asm `rep stosl` sẽ viết giá trị ở `eax` vào `edi`, tăng giá trị của thanh ghi `edi` lên `4` và giảm giá trị của thanh ghi `ecx` đi `1`. Lặp đi lặp lại thao tác này chừng nào giá trị ở thanh ghi `ecx` còn lơn hơn zero. Đó là tại sao lại đưa giá trị `6144` vào `ecx`.

`pgtable` được định nghĩa ở cuối file [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

Như chúng ta có thể thấy, nó sẽ nằm ở section `.pgtable` và kích thước của nó là `24` kilobytes.

Sau khi có được buffer cho cấu trúc `pgtable`, chúng ta có thể tạo bảng page cao nhất  - `PML4` - bằng:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Một lần nữa, chúng ta đặt địa chỉ tương đối của `pgtable` đến `ebx` hay chính là địa chỉ tương đối đến nhãn `startup_32` vào thanh ghi `edi`. Sau đó đặt địa chỉ này cộng thêm offset `0x1007` vào thanh ghi `eax`. Giá trị `0x1007` tức là `4096` bytes chính là kích thước của `PML4` cộng `7`. Số `7` biểu diễn cờ của `PML4` entry. Trong trường hợp của chúng ta, cờ này là `PRESENT+RW+USER`. Cuối cùng, chúng ta ghi địa chỉ đầu của  `PDP` entry đầu tiên vào `PML4`.

Bước tiếp theo, chúng ta sẽ tạo 4 entry cho `Page Directory` trong bảng `Page Directory Pointer` với giá trị cờ tương tự trên `PRESENT+RW+USE`:

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

Chúng ta đặt địa chỉ base của page directory pointer chính là vị trí có offset `4096` hay `0x1000` từ địa chỉ bảng `pgtable` vào thanh ghi `edi` và địa chỉ của page directory pointer entry đầu tiên vào thanh ghi `eax`. Đặt `4` vào thanh ghi `ecx`, nó sẽ trở thành biến đến trong vòng lặp dưới đây và viết địa chỉ của phần tử đầu tiên trong bảng page directory pointer table vào `edi` register. Sau bước này, `edi` sẽ chứa địa chỉ của page directory pointer entry đầu tiên, với giá trị cờ là `0x7`. Sau đó, chúng ta sẽ tính địa chỉ các các entries tiếp theo của bảng page directory pointer, mỗi cái `8` bytes, và viết địa chỉ của chúng vào `eax`. Bước cuối cùng trong việc tạo cấu trúc phân trang là tạo bảng trang có `2048` mỗi page là `2-MByte`:

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

Ở đây, chúng ta làm gần như giống hệ vị dụ trước, tất cả các entries được gán cờ - `$0x00000183` - `PRESENT + WRITE + MBZ`. Cuối cùng chúng ta có `2048` pages với kích thước `2-MByte` mỗi page:

```python
>>> 2048 * 0x00200000
4294967296
```

Tức là bảng trang có kích thước `4G`. Ban đầu chúng ta tạo một cấu trúc bảng trang cho phép biểu diễn `4` gigabytes bộ nhớ, giờ làm thế nào chúng ta đặt địa chỉ của các bảng high-level page table - `PML4` - trong thanh ghi điều khiển `cr3`:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

Đó là tất cả. Và tất cả các bước chuẩn bị đã kết thúc và giờ chúng ta sẽ xem quá trình chuyển sang long mode.

Chuyển sang 64-bit mode
--------------------------------------------------------------------------------

Đầu tiên nhất, chúng ta cần set cờ `EFER.LME` trong [MSR](http://en.wikipedia.org/wiki/Model-specific_register) về giá trị `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Ở đây, chúng ta đặt giá trị cờ `MSR_EFER` (được định nghĩa trong [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h#L7)) vào thanh ghi `ecx` và gọi lệnh asm `rdmsr`, lệnh này sẽ đọc thanh ghi [MSR](http://en.wikipedia.org/wiki/Model-specific_register). Sau khi lệnh `rdmsr` thực hiện, chúng ta sẽ có kết quả nằm ở `edx:eax`, giá trị này phụ thuộc vào giá trị `ecx`. Chúng ta sau đó kiểm tra giá trị bit `EFER_LME` bằng câu lệnh `btsl` và ghi dữ liệu từ thanh ghi `eax` vào thanh ghi `MSR` băng câu lệnh `wrmsr`.

Bước tiếp theo, chúng ta đặt code segment của nhân vào stack (cái chúng ta đã định nghĩa trong GDT) và đặt giá trị của hàm `startup_64` trong thanh ghi `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

Sau bước này, chúng ta sẽ đặt địa chỉ vào stack và bật phân trang bằng cách thiết lập bit `PG` và `PE` trong thanh ghi `cr0` register:

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

và thực hiện lệnh:

```assembly
lret
```

Nhớ rằng, chúng ta đã đẩy địa chỉ của hàm `startup_64` vào stack ở bước trước, và sau lệnh `lret`, CPU sẽ lấy địa chỉ đó và nhảy đến.

Sau tất cả những bước này, chúng ta cuối cùng đã nằm trong 64-bit mode:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

Đó là tất cả!

Kết luận
--------------------------------------------------------------------------------

Đây là đoạn cuối của phần 4 trong chương nói về quá trình boot nhân Linux. Nếu bạn có câu hỏi hoặc gọi ý nào, ping tôi [0xAX](https://twitter.com/0xAX) trên twitter, hoặc gửi [email](anotherworldofworld@gmail.com) hoặc đơn giản là tạo một [issue](https://github.com/0xAX/linux-insides/issues/new).

Trong phần tiếp theo, chúng ta sẽ xem quá trình giải nén nhân và sau đó ra sao.

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
