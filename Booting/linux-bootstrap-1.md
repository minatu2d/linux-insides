 Quá trình khởi động vào nhân. Phần 1.
================================================================================

 Từ bootloader đến nhân 
--------------------------------------------------------------------------------

 Nếu bạn đọc những bài viết trước đây trong [blog posts](http://0xax.blogspot.com/search/label/asm) của thôi, bạn có thể thấy rằng, thỉnh thoảng, tôi bắt đầu từ mức lập trình mức thấp (low-level programming). Tôi cũng viết một vài bài cho lập trình assembly x86_64 cho Linux. Cũng thời gian đó, tôi cũng bắt đầu lao vào tìm hiểu source code của Linux. Tôi thực sự có hứng thú với việc tìm hiểu về việc những thứ mức tập (low-level) làm việc như thế nào, làm thế nào một chương trình chạy được trên máy tính, chúng được xếp lên bộ nhớ như thế nào, làm thế nào nhân quản lý tiến trình (processes) và bộ nhớ (memory), các tầng mạng (network stack) làm việc ở mức thấp (low level), và rất nhiều thứ khác nữa. Chính vì thế, tôi quyết định viết một series bài viết này, cái liên quan đến nhân Linux kernel cho kiến trúc **x86_64**.

 Chú ý rằng là, tôi không phải một hacker nhân chuyên nghiệp, và tôi không viết code cho nhân trong công việc hàng ngày của mình. Nó đơn giản là thích thì tìm hiểu thôi (hobby). Tôi thích những thứ ở mức thấp (low-level stuff), và tôi quan tâm đến chúng làm việc như thế nào. Vì thế, nếu bạn gặp bất cứ sự khó hiểu nào trong các bài viết, hoặc có bất cứ câu hỏi, nhắc nhở nào, hãy ping tôi trên twitter [0xAX](https://twitter.com/0xAX), gửi đến địa chỉ [email](anotherworldofworld@gmail.com) hoặc đơn giản là tạo một [issue](https://github.com/0xAX/linux-insides/issues/new) trên github. Tôi đánh giá cao chúng. Tất cả các bài viết được lưu trữ trên github tại [linux-insides](https://github.com/0xAX/linux-insides), và cuối cùng nếu bạn có bạn tìm ra bất cứ lỗi sai về English trong lỗi dung bài viết, hãy thoải mái yêu cầu nha.


*Chú ý rằng, đây không phải là tài liệu chính thức, đây chỉ là học và chia sẻ lại kiến thức thôi.*

** Kiến thức yêu cầu để hiểu các bài viết **

* Hiểu code C
* Hiểu code assembly (dạng AT&T)

 Nào, nếu bạn sắn sàng để học thêm vài công cụ, tôi sẽ cố gắng giải thích chi tiết trong các bài viết sau đây. Đến đây thôi, là kết thúc phần giới thiệu đơn giản, giờ là lúc bắt đầu lao vào nhân (kernel) và những thứ bên dưới (low-level stuff).

 Hầu hết code của nhân được trích dẫn là phiên bản 3.18. Nếu có bất cứ sự thay đổi nào, tôi sẽ cập nhật trong bài viết tương ứng.

 Sau nút nguồn ma thuật (magic power button), chuyện gì xảy ra sau đó?
--------------------------------------------------------------------------------

 Mặc dù là series về nhân Linux, nhưng chúng ta sẽ chưa bắt đầu xem code nhân ngay - ít nhất là trong đoạn này. Ngay khi bạn nhấn nút nguồn trên máy tính xách tay hoặc máy để bàn, thì chúng sẽ bắt đầu chạy. Bo mạch chủ (motherboar) gửi tín hiệu đến bộ cấp nguồn [power supply](https://en.wikipedia.org/wiki/Power_supply). Sau khi nhận được tín hiệu, bộ cấp nguồn sẽ cung cấp một lượng điện phù hợp cho máy tính. Một khi bo mạch chủ nhận được [power good signal](https://en.wikipedia.org/wiki/Power_good_signal), nó sẽ cố gắng khởi động CPU. CPU sẽ reset tất cả các thanh ghi của nó, rồi đưa giá trị mặc định (đã được định nghĩa trước) vào.


 Bộ xử lý [80386](https://en.wikipedia.org/wiki/Intel_80386) và các CPU sau này có một bộ các giá trị định nghĩa trước cho các thanh ghi sau khi máy tính được reset:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

Tiếp đó, CPU sẽ bắt đầu làm việc ở [real mode](https://en.wikipedia.org/wiki/Real_mode). Chúng ta dừng lại một chút để hiểu cấu trúc bộ nhớ (understand memory segmentation) ở mode này. Mode thực (Real mode) được hỗ trợ trên tất cả các bộ xử lý tương thích với x86 (x86-compatible), từ [8086](https://en.wikipedia.org/wiki/Intel_8086) cho đế các bộ xử lý 64-bit hiện tại của Intel 64-bit. Bộ xử lý 8086 có bus địa chỉ độ dài 20-bit, có nghĩa là chúng có thể làm việc trong không gian địa chỉ từ 0-0xFFFFF (tức là 1 megabyte). Nhưng thanh ghi chỉ có độ dài 16-bit, tức là địa chỉ lớn nhất chứa có thể chứa trong 1 thanh ghi là 2^16 - 1 hay 0xffff ( tức 64 kilobytes). [Memory segmentation](http://en.wikipedia.org/wiki/Memory_segmentation) được sử dụng để tận dụng hết không gian địa chỉ có thể có. Toàn bộ bộ nhớ sẽ được chia nhỏ ra, thành các đoạn (segments) có độ dài 65536 bytes ( hay 64 KB). Vì chúng ta không thể đánh địa chỉ bộ nhớ lớn hơn 64 KB  với thanh ghi độ dài 16 bit được, và giải phải như vậy đã được nghĩ ra. Một địa chỉ sẽ chứa 2 phần: segment (segment selector), sử dụng để tính địa chỉ cơ bản (base address), phần bù (offset) tính từ vị trí của địa chỉ cơ bản. Trong chế độ thực (real mode), địa chỉ cơ bản (base address) tương ứng vớisegment (segment selector) được tính bằng `Giá trị Segment Selector * 16`. Vì thế, để tính địa chỉ thực trên bộ nhớ, ta cần nhân giá trị segment với 16 rồi cộng với giá trị phần bù (offset):

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Ví dụ, nếu CS (giá trị segment), và IP (giá trị offset) có giá trị tương ứng `CS:IP` bằng `0x2000:0x0010`, thì địa chỉ vật lý tương ứng sẽ được tính như sau:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Nhưng, nếu ta lấy giá trị lớn nhất của cả segment selector và offset, `0xffff:0xffff`, thì địa chỉ tính được sẽ như sau:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

Tức là kéo dài đến 65520 sau Megabyte đầu tiên. Vì chỉ được phép truy cập 1MB trong real mode, phần địa chỉ tính tđến `0x10ffef` có độ dài `0x00ffef` sẽ bị vô hiệu hóa [A20](https://en.wikipedia.org/wiki/A20_line).

Ok, giờ chúng ta đã biết về real mode và cách đánh địa chỉ bộ nhớ (memory addressing). Nào, cùng quay trở lại thảo luận về giá trị thanh ghi sau khi CPU được reset:

Thanh ghi `CS` đúng ra chứa 2 phần: phần thấy được segment selector, và địa chỉ cơ bản (base address) ẩn. Địa chỉ cơ bản thông thương được tính bằng tích của segment selector cho 16, trong quá trình reset, thì giá trị segment selector trong thanh ghi CS được đưa vào giá trị 0xf000 và địa chỉ cơ bản (base address) được coi là 0xffff0000; bộ xử lý sẽ sử dụng giá trị địa chỉ đặc biệt này cho đến khi giá trị thanh ghi `CS` bị thanh đổi.

 Địa chỉ bắt đầu được tính bằng cách cộng địa chỉ cơ bản với giá trị trong thanh ghi EIP:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

 Chúng ta sẽ có giá trị địa chỉ `0xfffffff0`, địa chỉ này bắt đầu cho đoạn 16 bytes cuối của 4GB đầu tiên. Điểm này được gọi là [Reset vector](http://en.wikipedia.org/wiki/Reset_vector). Đây là chỉ trên bộ nhớ mà CPU sẽ thực hiện chạy câu lệnh máy đầu tiên sau khi nó reset. Ở đó, chứa một lệnh [jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (kí hiệu là `jmp`), thông thường sẽ trỏ đến điểm đầu vào (vị trí đầu tiên chứa mã lệnh) của BIOS.  Ví dụ, nếu nhìn vào mã nguồn của [coreboot](http://www.coreboot.org/), chúng ta có thể thấy:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

Như ở trên, chúng ta có thấy lệnh `jmp` với mã [opcode](http://ref.x86asm.net/coder32.html#xE9) là 0xe9, lệnh này sẽ nhảy đến địa chỉ `_start - ( . + 2)`. Phần code đoạn `reset` này là 16 bytes, và nó bắt đầu ở vị trí `0xfffffff0`:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

 Bây giờ thì, BIOS được chạy; Sau khi thực hiện khởi tạo và kiểm tra phần cứng, BIOS cần tìm một thiết bị có thể kboot được (bootable device). Thứ tự boot được lưu trong cấu hình BIOS (BIOS configuration), điều khiển thiết thứ tự các thiết bị được BIOS tìm và thực hiện boot. Khi thực hiện boot từ ổ đĩa cứng (hard drive), BIOS sẽ cố gắng tìm một boot sector. Trên ổ đĩa cứng, thường có một bảng gọi là MBR, chứa vị trí các phân vùng, nằm ở phần được gọi là boot sector là 446 bytes của sector đầu tiên, mỗi sector sẽ có 512 bytes. 2 byte cuối của sector đầu tiên (tưc là 510, 511) là `0x55` và `0xaa`, cái sẽ cho BIOS biết rằng đây thiết bị chứa sector đó có thể boot được. Ví dụ:

```assembly
;
; Chú ý: Mã của ví dụ này viết theo cú pháp Assembly của Intel
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Build và chạy source trên bằng câu lệnh sau:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

 Câu lệnh này sẽ bảo [QEMU](http://qemu.org) hãy sử dụng file nhị phân có tên là `boot` như là một ảnh đĩa (hay là ổ cứng của nó). Vì binary được sinh ra từ đoạn code Assembly ở trên, nên nó sẽ thỏa mãn yêu cầu về boot sector ( điền tất cả bằng giá trị `0x7c00`, cuối cùng là chuỗi ma thuật 0x55, 0xaa (magic sequence)), QEMU sẽ "bị hiểu" đoạn đó như là MBR (master boot record) của ảnh đĩa (disk image).

Bạn sẽ thấy ở hình dưới đây:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

Trong ví dụ này, chúng ta thấy rằng, code được chạy trong chế 16 bit real mode và bắt đầu từ địa chỉ `0x7c00` trên bộ nhớ. Sau khi bắt đầu chạy, nó gọi ngắt [0x10](http://www.ctyme.com/intr/rb-0106.htm), cái đơn giản chỉ để in ra kí tự  `!`; 510 bytes tiếp theo được điền hết là zero, cuối cùng là 2 byte ma thuật `0xaa` and `0x55`.

Bạn có thể xem nội dung đoạn binary được nói ở trên bằng công cụ có tên `objdump`:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

 Một boot sector thật sẽ code xử lý để tiếp tục quá trình boot rồi một bảng phân vùng thay vì một mớ giá trị zero, và dấu chấm cảm như ở trên :) Từ thời điểm này, BIOS sẽ trao quyền điều khiển cho bootloader.

**NOTE**: Như đã giải thích ở trên, CPU nằm trong real mode; mà trong real mode, việc tính toán địa chỉ bộ nhớ được thực hiện theo công thức sau:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

 Như đã giải thích ở phía trên rồi. Chúng ta chỉ có những thanh ghi 16 bit mà thôi; giá trị của một thanh ghi 6 bit là `0xffff`, vì thế, giá trị địa chỉ lớn nhất mà thanh ghi có thể miêu tả:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

Giá trị `0x10ffef` tương đương với `1MB + 64KB - 16b`. Một bộ xử lý [8086](https://en.wikipedia.org/wiki/Intel_8086) (là bộ xử lý đầu tiên được có real mode), thế nhưng ngược lại, chỉ có 20 đường bit địa chỉ mà thôi. Vì `2^20 = 1048576` bằng 1MB, điều này có nghĩa rằng số bộ nhớ thực sự nó có thể sử dụng là 1MB.

 Bản đồ phân chia bộ nhớ (memory map) trong real mode được miêu tả như sau:

```
0x00000000 - 0x000003FF - Bảng ngắt trong chế độ Readl Mode (Real Mode Interrupt Vector Table)
0x00000400 - 0x000004FF - Vùng dữ liệu của BIOS (BIOS Data Area)
0x00000500 - 0x00007BFF - Không sử dụng (Unused)
0x00007C00 - 0x00007DFF - Dành cho bootloader (Our Bootloader)
0x00007E00 - 0x0009FFFF - Không sử dụng (Unused)
0x000A0000 - 0x000BFFFF - RAM dành cho Video (Video RAM (VRAM) Memory)
0x000B0000 - 0x000B7777 - Bộ nhớ dành cho chế độ Video đơn sắc (Monochrome Video Memory)
0x000B8000 - 0x000BFFFF - Bộ nhớ dành cho video màu (Color Video Memory)
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - BIOS hệ thống (System BIOS)
```

Trong đoạn mở đầu của bài này, tôi đã viết rằng, lệnh đầu tiên được CPU chạy nằm ở địa chỉ `0xFFFFFFF0`, tức là nó còn lớn hơn của `0xFFFFF` (1MB). Vậy thì làm thế nào CPU truy cập được địa chỉ này trong chế độ real mode? Câu trả lời nằm ở tài liệu của [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte của ROM được map vào không gian địa chỉ (128 kilobyte ROM mapped into address space)
```

Ở thời điểm bắt đầu chạy, BIOS không nằm trên RAM, mà nằm trên ROM.

Bootloader
--------------------------------------------------------------------------------

 Có một cơ số các bootloader có thể boot Linux, chẳng hạn [GRUB 2](https://www.gnu.org/software/grub/) và [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). Nhân Linux có một cái gọi là [Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt), cái quy định những yêu cầu dành cho một bootloader khi định hỗ trợ Linux. Ví dụ này sẽ miêu tả về GRUB2.

 Tiếp tục của nội dung bên trên, làm thế nào BIOS chọn thiết bị boot và chuyển quyền điều khiển (transferred control) cho đoạn code được đặt trong boot sector (boot sector code), việc chạy được bắt đầu từ [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). Đoạn code này rất đơn giản, vì sự giới hạn của không gian trống trong bộ nhớ, nó chỉ chứa một pointer nhảy đến vị trí của core image của GRUB 2 (GRUB 2's core image). Các core image này bắt đầu bằng [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), thường được lưu ngay sau sector đầu tiên của vùng trống không sử dụng cho đến trước phần vùng đầu tiên. Đoạn code chứa trong phần đầu của core image (diskboot.img) core image sẽ thực hiện load phần còn lại vào bộ nhớ, cái phần mà chứa nhân của GRUB 2's kernel các drivers để hiểu hệ thống file (filesystems). Sau khi load phần còn lại này, nó sẽ chạy (nhảy đến) [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c).

 Hàm `grub_main` khởi tạo console ( tức là cái cho phép nhận phím, hiển thị màn hình mà ta vẫn thấy), lấy thông tin địa chỉ của các module, thiết lập thiết bị root (root device), đọc/hiểu (loads/parses) file cấu hình grub, load các module vào, etc. Ở xử lý cuối của hàm, `grub_main` chuyển sang chạy ở chế độ normal mode. `grub_normal_execute` (from `grub-core/normal/main.c`) để coi như hoàn thành bước chuẩn bị cuối cùng, sau đó hiển thị menu cho phép chọn hệ điều hành. Khi chúng ta chọn một trong những menu đó, hàm `grub_menu_execute_entry` được chạy, thực hiện lệnh `boot` của GRUB để boot vào hệ điều hành được chọn.

 Bạn có thể đọc trong giao thức khởi động nhân Linux (kernel boot protocol), bootloader phải đọc/điền giá trị cho một số trường trong thông số thiết lập kernel (kernel setup header), cái sẽ nằm ở vị trí có offset `0x01f1` tính từ phần đầu của đoạn code thiết lập kernel. Header dành cho kernel [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) được mô tả như sau:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

Bootloader có nhiệm vụ điền các giá trị thấy ở trên, còn các giá trị khác ( cái được đánh dấu có kiểu `write` trong giao thức boot Linux, như trong [ví dụ](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)) này chẳng hạn, giá trị hoặc được nhận từ tham số truyền hoặc phải được tính. (Chúng ta không đi vào chi tiết các trường của phần header thiết lập nhân, thay vào đó chúng ta sẽ xem xét nhân sử dụng những giá trị đó như thế nào; bạn có thể tìm hiểu chi tiết các trường đó tại tài liệu [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156).)

Như bạn thấy trong giao thức boot nhân, cấu trúc bộ nhớ (memory map) sẽ được sắp xếp dưới đâu sau khi load nhân:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Phần không sử dụng sẽ còn lại nhiều nhất có thể 
         ~                        ~
         | Command line           | (Có thể nằm bên dưới vị trí X+10000)
X+10000  +------------------------+
         | Stack/heap             | Được sử dụng bởi code nhân ở chế độ real-mode.
X+08000  +------------------------+
         | Kernel setup           | Code của nhân khi chạy ở chế độ real-mode.
         | Kernel boot sector     | Nhân cho loại boot sector cũ.
       X +------------------------+
         | Boot loader            | <- Điểm vào của boot sector vị trí 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

Dựa vào sơ đồ trên, khi bootloader chuyển quyền điều khiển cho nhân, nó sẽ bắt đầu ở vị trí:

```
X + sizeof(KernelBootSector) + 1
```

Ở đây, `X` là địa chỉ mà sector boot nhân được load vào. Trong máy của tôi, `X` bằng `0x10000`, chúng ta có thể thấy trong memory dump bên dưới đây:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

Bootloader giờ sẽ load nhân Linux vào bộ nhớ, điền giá trị các trường header, sau đó nhảy đến vị trí bộ nhớ tương ứng. Chúng ta có thể di chuyển trực tiếp đến đoạn code thiết lập nhân.

 Bắt đầu của việc thiết lập nhân (Start of kernel setup)
--------------------------------------------------------------------------------

 Cuối cùng thì, chúng ta đã ở trong nhân rồi! Về mặt kĩ thuật, thì nhân vẫn chưa được chạy đâu; Đầu tiên, chúng ta cần thiết lập nhân, bộ quản lý bộ nhớ, bộ quản lý tiến trình, etc. Đoạn code chạy thiết lập nhân bắt đầu từ [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) ở hàm [_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293). Ban đầu trong nó hơi lạ, vì có một vài mã lệnh phía trước.

Một thời gian dài trước kia, nhân Linux sử dụng bootloader riêng cho nó. Bây giờ, nếu bạn ví dụ như sau chẳng hạn,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

Bạn sẽ thấy:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

 Thực sự thì, `header.S` bắt đầu từ [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (như hình bên trên), một dòng in ra lỗi và header của [PE](https://en.wikipedia.org/wiki/Portable_Executable):

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

Đoạn này là cần thiết khi load các hệ điều hành được khởi động theo chuẩn [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface). Chúng ta sẽ không xem xét xử lý bên trong của nó ngay bây giờ, nó sẽ được nói đến ở những chương tiếp sau.

Đoạn thiết lập nhân thực sự bắt đầu từ đoạn sau:

```assembly
// header.S line 292
.globl _start
_start:
```

Các bootloader ( như: grub2 và những cái tương tự) biết rõ điểm bắt đầu này ( vị trí offset `0x200` tính từ `MZ`)  nên nó sẽ thực hiện một thao tác nhảy (jump) đến đó, mặc dù file header `header.S` bắt đầu từ đoạn `.bstext` để in ra một message lỗi rồi:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

Điểm đầu vào thiết lập nhân:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

 Ở đây chúng ta thấy mã lệnh của lệnh `jmp` (`0xeb`), thực hiện nhảy đến điểm có nhãn `start_of_setup-1f`. Bằng kí pháp `Nf`, `2f` tham chiếu đến vị trí cục bộ có nhãn `2:`; trong trường hợp này, đó là nhãn `1`, cái ở ngay sau lệnh jump như ta thấy ở trên, đoạn này sẽ chứa phần còn lại của code thiết lập trong [header](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156). Ngay sau đoạn thiết lập này, chúng ta sẽ thấy đoạn `.entrytext`, cái sẽ bắt đầu đoạn có nhãn `start_of_setup`.

 Đây mới là mã lệnh đầu tiên thực sự được chạy ( tất nhiên có thể tính thêm cả lệnh jump trước đó). Sau khi đoạn code thiết lập nhân nhận điều khiển từ bootloader, lệnh `jmp` sẽ nhảy đến vị trí có offset là `0x200` từ vị trí của đoạn code nhân trong chế độ real mode, ví dụ sau 52 byte đầu tiên. Ở đây chúng ta có thể đọc giao thức boot nhân Linux và kiểm tra trên source code của grub2:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

 Đoạn này sẽ nói lên rằng các thanh ghi segment sẽ nhận các giá trị như dưới đây sau khi thiết lập nhân bắt đầu:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

 Trong trường hợp của tôi, nhân được load ở địa chỉ `0x10000`.

 Sau khi nhảy đến nhãn `start_of_setup`, đầu tiên nhân sẽ làm những việc sau:

* Kiểm tra để chắc chắn rằng các thanh ghi segment có cùng giá trị 
* Thiết lập lại stack nếu cần 
* Thiết lập [bss](https://en.wikipedia.org/wiki/.bss)
* Nhảy đến đoạn mã được sinh ra bởi code C trong [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)

 Nào chúng ta hãy xem nó được viết như thế nào.

Segment registers align
--------------------------------------------------------------------------------

 Đầu tiên nhất, nhân sẽ đảm bảo giá trị thanh ghi segment `ds` và `es` cùng trỏ đến một địa chỉ. Tiếp theo, nó clear giá trị các cờ hướng sử dụng lệnh  `cld`:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

 Như đã viết trước đó, grub2 load đoạn code thiết lập nhân vào địa chỉ `0x10000` và `cs` sẽ được thiết lập trỏ đến địa chỉ `0x1020` bởi vì quá trình chạy không thể bắt đầu từ vạch bắt đầu được, nhưng từ đoạn

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

lệnh `jump`, sẽ nhảy đến vị trí cách 512 byte từ đoạn chứ giá trị magic [4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47). Bạn cần điều chỉnh thanh ghi `cs` từ giá trị `0x10200` sang giá trị `0x10000`, các thanh ghi khác cũng thế. Sau đó, chúng ta thiết lập lại stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

 sẽ thực hiện đẩy giá trị `ds` chứa địa chỉ của nhãn là [6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494), rồi chạy lệnh `lretw`. Khi lệnh `lretw` này được gọi, nó sẽ load địa chỉ của nhãn `6` vào thanh ghi [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) và load giá trị cho thanh ghi  `cs` bằng chính giá trị của thanh ghi `ds` trước đó. Hay nói cách khác, sau bước này, thanh ghi `ds` và `cs` sẽ có cùng giá trị.

Thiết lập stack
--------------------------------------------------------------------------------

 Hầu hết code trong phần thiết lập chạy để chuẩn bị cho môi trường chạy mã C (mã máy sinh ra từ các chương trình C) trong real mode. [Bước tiếp theo đó](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467)  là thực hiện kiểm tra giá trị thanh ghi `ss` và sửa lại cho đúng nếu nó bị sai:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

Điều này dẫn đến 3 ngữ cảnh khác nhau:

* `ss` chứa giá trị hợp lệ 0x10000 ( giống như tất cả các thanh ghi khác tương tự `cs`)
* `ss` chứa giá trị không hợp lệ và cờ `CAN_USE_HEAP` được bật     (xem thêm bên dưới)
* `ss` chứa giá trị không hợp lệ và cờ `CAN_USE_HEAP` không được bật (xem thêm bên dưới)

 Chúng ta sẽ cùng nhìn 3 ngữ cảnh này theo thứ tự nào:

* `ss` chứa giá trị hợp lệ (0x10000). Trong trường hợp này , chúng ta sẽ đến nhãn là [2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

 Ở đây ta thấy, 4 bytes kiểm tra được sử dụng để kiểm tra xem giá trị `dx` ( chứa giá trị `sp` được cung cấp bởi bootloader) có bằng 0 hay không. Nếu nó bằng zero, chúng ta sẽ gán giá trị `0xfffc` ( dịch chuyển 4 byte trước địa chỉ của số byte lớn nhất 64 KB) trong `dx`. Nếu nó khác zero, chúng ta tiếp tục sử dụng giá trị `sp`, được gán bởi bootloader ( trong trường hợp của tôi là 0xf7f4). Sau bước này, chúng ta sẽ đặt giá trị `ax` thanh ghi `ss`, chính là thanh ghi lưu địa chỉ `0x10000` và thiết lập đúng cho `sp`. Giờ chúng ta có một stack hợp lệ rồi:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* Trong ngữ cảnh thứ 2, ( giá trị `ss` != `ds`). Đầu tiên, chúng ta đặt giá trị của [_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52) (địa chỉ cuối cùng của đoạn code thiết lập) vào thanh ghi `dx`  rồi kiểm tra giá trị cờ `loadflags` sử dụng lệnh `testb` instruction, để xem có thể sử dụng bộ nhớ heap không. Các giá trị của [loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) là các cờ bitmask được định nghĩa ở đây:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

và, bạn cũng thể đọc nó trong giao thức boot với đoạn như sau,

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

Nếu giá trị cờ `CAN_USE_HEAP` được bật, chúng ta sẽ phải đặt giá trị `heap_end_ptr` vào thanh ghi `dx` ( cái đang trỏ đến `_end`) và cộng giá trị kích thước `STACK_SIZE` ( kích thước nhỏ nhất là 512 bytes) vào đó. Sau bước này,  nếu giá trị `dx` được nhớ lại ( nó sẽ không được nhớ lại, dx = _end + 512), chúng ta sẽ nhảy đến nhãn `2` (như trong trường hợp đầu tiên) và đảm bảo stack hợp lệ.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* Khi mà cờ `CAN_USE_HEAP` không được set, chúng ta đơn giản sử dụng một stack với kích thước nhỏ nhất từ `_end` đến `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

Thiết lập BSS
--------------------------------------------------------------------------------

 Hai bước cuối cùng cần được thực hiện trước khi nhảy đến hàm main của code C là thiết lập vùng [BSS](https://en.wikipedia.org/wiki/.bss) và kiểm tra chữ kí "magic" ("magic" signature). Đầu tiên, kiểm tra chữ kí:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

Đoạn này đơn giản là so sánh giá trị [setup_sig](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L39) với một dãy số magic `0x5a5aaa55`. Nếu chúng không bằng nhau, lỗi không thể tiếp tục (fatal error) sẽ được thông báo.

 Nếu đoạn chúng bằng nhau, nó sẽ cho chúng ta biết rằng chúng ta đã thiết lập đúng các thanh ghi segement (segment registers) và stack, bạn cần thiết lập vùng BSS trước khi nhảy đến code C.

 Vùng BSS được sử dụng cho việc cấp phát tĩnh, chứa các giá trị cho biến chưa được khởi tạo. Linux đảm bảo một cách cẩn thận rằng vùng này đầu tiên sẽ được đưa hết về zero thông qua đoạn code sau:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

 Đầu tiên, địa chỉ của [__bss_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L47) được đưa vào thanh ghi `di`. Sau đó, giá trị ở địa chỉ `_end + 3` (+3 - tiến 4 bytes) được lưu vào `cx`. Thanh ghi `eax` xóa (gán về 0) ( sử dụng lệnh `xor`), kích thước của bss (`cx`-`di`) được tính rồi đưa vào thanh ghi `cx`. Rồi sau đó, giá trị ở thanh ghi `cx` được chia cho 4 ( bằng kích thước 1 word 'word'), lệnh `stosl` được thực hiện lặp lại, lưu giá trị ở thanh ghi `eax` ( hiện tại là zero) vào địa chỉ chứa ở thanh ghi `di`, ngay sau đó giá trị ở thanh ghi `di` sẽ được tăng lên 4, lặp lại quá trình này đến khi `cx` bằng 0). Mục đích chính của quá trình này là ghi zero vào tất cả các vị trí nhớ từ `__bss_start` cho đến `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Nhảy đến main
--------------------------------------------------------------------------------

 Rồi, tất cả những gì ta có là - stack và BSS, giờ có thể nhảy đến hàm `main()` được sinh ra từ code C được rồi:

```assembly
    calll main
```

 Cái hàm `main()` có source code ở [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). Bạn có thể biết chúng thực hiện gì ở đó trong chương sau.

Kết luận
--------------------------------------------------------------------------------

 Đây là đoạn cuối trong chương đầu tiên về Linux kernel insides. Nếu bạn có bất cứ câu hỏi hoặc gợi ý nào, hãy ping cho tôi trên [0xAX](https://twitter.com/0xAX), gửi [email](anotherworldofworld@gmail.com), hoặc đơn giản là tạo một [issue](https://github.com/0xAX/linux-internals/issues/new). Trong chương tới, chúng ta sẽ xem đoạn code C đầu tiên được chạy khi thiết lập nhân Linux, các thực thi (implementation) của các hàm cơ bản như `memset`, `memcpy`, `earlyprintk`, thực thi của console sớm (early console implementation) và cả khởi tạo nữa, cũng như những thứ khác.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
