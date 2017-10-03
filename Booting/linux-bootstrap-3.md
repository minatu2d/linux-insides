Quá trình khởi động nhân. Phần 3 
================================================================================

Khởi tạo chế độ Video mode và chuyển sang protected mode
--------------------------------------------------------------------------------

Đây là phần 3 trong series nói về `Quá trình khởi động nhân`. Trong [part](linux-bootstrap-2.md#kernel-booting-process-part-2) trước, chúng ta đang dừng lại trước lời gọi hàm `set_video` trong [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L181). Trong phần này, chúng ta sẽ xem:
- khởi tạo  video mode trong code setup của kernel,
- các chuẩn bị trước khi chuyển sang protected mode,
- chuyển sang protected mode

**Chú ý** Nếu bạn chưa biết gì về protected mode, bạn có thể tìm thấy thông tin về nó ở [part](linux-bootstrap-2.md#protected-mode) trước. Cũng có một danh sách [links](linux-bootstrap-2.md#links) tham khảo mà có thể giúp ích cho bạn.

Như đã viết ở trên, chúng ta sẽ bắt đầu từ hàm `set_video` được định nghĩa ở file source [arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/video.c#L315). Chúng ta thấy rằng, nó bắt đầu bằng việc đọc thiết lập video mode từ cấu trúc `boot_params.hdr`:

```C
u16 mode = boot_params.hdr.vid_mode;
```

Chính là giá trị chúng ta đã điền trong hàm `copy_boot_params`( bạn có thể đọc về nó ở phần trước của series). `vid_mode` là trường bắt buộc được điền bởi bootloader. Bạn có thể tìm thấy thông tin về nó ở đặc tả giao thức khởi động nhân:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

Như đoạn được trích từ giao thức khởi động nhân:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

Vì thế, bạn có thể thêm tham số `vga` vào file cấu hình của grub hoặc một bootloader khác, nó sau đó sẽ truyền các tham số này đến dòng lệnh của nhân. Tham số này có thể nhận các giá trị khác giống như phần miêu tả ở trên có nói. Ví dụ, nó có thể là một số nguyên dạng `0xFFFD` hoặc `ask`. Nếu truyền vào giá trị `ask` cho `vga`, bạn sẽ thấy menu giống như thế này:

![video mode setup menu](http://oi59.tinypic.com/ejcz81.jpg)

Nó sẽ hỏi để chọn một video mode. Chúng ta cùng xem xét bên trong hàm, nhưng để hiểu tốt hơn trước chúng ta sẽ tìm hiểu một vài thức khác.

Kiểu dữ liệu trong nhân
--------------------------------------------------------------------------------

Ở các phần trước, chúng ta đã nhìn thấy các kiểu dữ liệu được sử dụng kiểu như `u16` chẳng hạn trong code thiết lập nhân. Chúng ta cùng xem một loại kiểu dữ liệu được sử dụng trong nhân:


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

Nếu bạn đọc code của nhân, bạn sẽ thấy chúng rất thường xuyên, tốt hơn chúng ta nên nhớ chúng.

API cho Heap
--------------------------------------------------------------------------------

Sau khi lấy giá trị `vid_mode` từ `boot_params.hdr` trong hàm `set_video`, Chúng ta có thể thấy một lời gọi đến hàm thông qua macro `RESET_HEAP`. `RESET_HEAP` là macro được định nghĩa ở [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h#L199). Định nghĩa của nó như sau:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

Nếu bạn đã đọc phần 2 của series này, bạn sẽ nhớ ra rằng chúng ta đã khởi tạo heap bằng hàm [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116). Chúng ta cũng có một nhóm các hàm dành cho thao tác với  heap, chúng được định nghĩa ở `boot.h`. Đó là:

```C
#define RESET_HEAP()
```

Như chúng ta thấy ở trên, nó reset bộ nhớ heap bằng cách gán giá trị biến `HEAP` bằng `_end`, `_end` ở đây đơn giản được lấy từ định nghĩa `extern char _end[];`

Tiếp theo đó là `GET_HEAP` macro:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

để cấp phát bộ nhớ heap. Macro này gọi một hàm khác có tên `__get_heap` với 3 tham số gồm:

* kích thước của kiểu (theo bytes) cho mỗi phần tử
* `__alignof__(type)` chỉ ra biến này được `aligned` như thế nào 
* `n` số phần tử sẽ cấp phát

Xử lý của hàm `__get_heap` như sau:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

và cách hàm sử dụng của nó, giống như sau:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

Hãy cùng nhau xem xét `__get_heap` hoạt động như thế nào. Chúng ta thấy ở đây, giá trị `HEAP` ( được gán bằng `_end` sau hàm `RESET_HEAP()`) chính là địa chỉ ô nhớ (đã aligned) của tham số `a`. Sau bước này, chúng ta sẽ lưu địa chỉ của `HEAP` sang biến `tmp`, rồi di chuyển `HEAP` về cuối của block được cấp phát và trả về giá trị `tmp` chính là địa chỉ bắt đầu của ô nhớ được cấp phát.

Và hàm cuối cùng trong nhóm liên quan đến heap là:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

ở đây một lượng `HEAP` được trừ đi từ `heap_end` ( như đã tính toán ở [part](linux-bootstrap-2.md) trước) và trả về giá trị 1 nếu nếu bộ nhớ còn trống lơn hơn `n`.

Tất cả rồi đó. Giờ chúng ta đã có một nhóm API cho heap rồi, giờ có thể đến phần thiết lập video mode.

Thiết lập video mode
--------------------------------------------------------------------------------

Bây giờ chúng ta chuyển sang phần khởi tạo  video mode được rồi. Trước đó, chúng ta đang dừng tại lời gọi hàm `RESET_HEAP()` trong hàm  `set_video`. Tiếp theo đó đến lời gọi `store_mode_params` để lưu các tham số dành cho video mode có trong trường `boot_params.screen_info`, được định nghĩa tại [include/uapi/linux/screen_info.h](https://github.com/0xAX/linux/blob/master/include/uapi/linux/screen_info.h).

Hãy xem hàm `store_mode_params`, chúng ta thấy rằng, nó bắt đầu bằng lời gọi hàm `store_cursor_position`. Từ tên của hàm này, bạn cũng có thể hiểu được, nó lấy thông tin con trỏ màn hình và lưu vào đó.

 Trong hàm `store_cursor_position`, đầu tiên nó khởi tạo 2 biến có kiểu là `biosregs` với giá trị `AH = 0x3`, và thực hiện gọi ngắt BIOS `0x10`. Sau khi ngắt được thực hiện thành công, giá trị hàng và cột của con trỏ sẽ được lư ở thanh ghi `DL` và `DH`. Hàng và cột được lưu ở 2 trường `orig_x` và `orig_y` trong cấu trúc `boot_params.screen_info`.

 Sau khi hàm `store_cursor_position` được chạy, đến hàm `store_video_mode` sẽ được gọi. Nó đơn giản là lấy giá trị hiện tại của video mode và lưu vào trong `boot_params.screen_info.orig_video_mode` mà thôi. 

Tiếp đó, nó sẽ kiểm tra video mode hiện tại và gán giá trị `video_segment`. Sau khi BIOS truyền điều khiển đến boot sector, những địa chỉ sau sẽ dành cho video memory:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

 Vì thế, chúng ta sẽ gán giá trị biến `video_segment` bằng `0xB000` nếu video mode hiện tại là MDA, HGC, hoặc VGA trong monochrome mode (đen trắng) và bằng giá trị `0xB800` nếu video mode hiện tại trong color mode (chế độ màu). Sau khi thiết lập địa chỉ của video segment, kích thước font cũng phải được lưu vào `boot_params.screen_info.orig_video_points` bằng đoạn lệnh sau:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

 Đầu tiên, chúng ta đặc giá trị 0 vào thanh ghi `FS` bằng hàm `set_fs`. Chúng ta lại thấy hàm `set_fs` đã được nhắc đến ở phần trước. Tất cả chúng được định nghĩa trong [boot.h](https://github.com/0xAX/linux/blob/master/arch/x86/boot/boot.h). Tiếp theo, chúng ta đọc các giá trị được lưu ở địa chỉ `0x485` (địa chỉ này được sử dụng để lấy kích thước font chữ), rồi lưu vào trường `boot_params.screen_info.orig_video_points`.

```
 x = rdfs16(0x44a);
 y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

Tiếp theo, chúng ta sẽ lấy số cột tại địa chỉ `0x44a` và số dòng tại địa chỉ `0x484`, sau đó lưu chúng vào `boot_params.screen_info.orig_video_cols` và `boot_params.screen_info.orig_video_lines` tương ứng. Đến đây, hàm `store_mode_params` kết thúc.

Tiếp theo, chúng ta sẽ thấy hàm `save_screen`, đơn giản là lưu nội dung màn hình vào heap. Hàm này sẽ tập hợp các thông tin ta lấy ở hàm trước đó như số hàng, cột etc. và lưu vào cấu trúc tên là `saved_screen`, được định nghĩa như bên dưới đây:

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

Sau đó, nó kiểm tra heap xem đủ vùng trống cho nó không:

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

rồi xin cấp phát nếu đủ, sau đó lưu giá trị kiểu cấu trúc `saved_screen` vào đó.

Lời gọi hàm tiếp theo là `probe_cards(0)` được định nghĩa ở [arch/x86/boot/video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L33). Nó xem thông tin video_cards và lấy các mode được cung cấp bởi các card này. Đây là một đoạn khá thú vị, chúng ta có thể thấy vòng lặp như sau:

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* tập hợp thông tin các mode - collecting number of modes here */
}
```

nhưng ở đây `video_cards` chưa được định nghĩa ở đâu hết. Câu trả lời khá đơn giản : Mỗi video mode biểu diễn trong code thiết lập nhân dành cho cấu trúc x86 được định nghĩa như sau:

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

ở đây, `__videocard` là một macro có định nghĩa như sau:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

Cấu trúc `card_info` được định nghĩa như sau:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

định nghĩa bằng macro ở trên sẽ nằm trong segment `.videocards`. Cùng xem file script dành cho linker [arch/x86/boot/setup.ld](https://github.com/0xAX/linux/blob/master/arch/x86/boot/setup.ld), chúng ta có thể thấy:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

Cái này có nghĩa rằng `video_cards` đơn giản là một địa chỉ trên bộ nhớ và tất cả các biến cấu trúc `card_info` sẽ được đặt trong segment này. Có nghĩa là tất cả các biến cấu trúc `card_info` được đặt ở giữa `video_cards` và `video_cards_end`, vì thế chúng ta có thể sử dụng một vòng lặp để đi hết qua chúng. Sau khi hàm `probe_cards` chạy xong, chúng ta sẽ có tất cả các cấu trúc dạng `static __videocard video_vga` được gán `nmodes` ( số lượng video modes tương ứng).

Sau đó khi chạy hàm `probe_cards` xong, chúng ta sẽ bị quay trở lại vòng lặp chính trong hàm `set_video`. Trong đó, có một vòng lặp vô hạn để thiết lập video mode bằng hàm `set_mode`, hoặc in ra menu nếu kernel được truyền tham số `vid_mode=ask` hoặc không xác định được video mode. 

Cái hàm `set_mode` được định nghĩa trong file mã nguồn [video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L147) và chỉ nhận 1 tham số, đó là `mode`, đó chính là số lượng video modes ( cái chúng ta nhận được từ menu hoặc trong đoạn đầu của hàm `setup_video`, từ header thiết lập nhân). 

Hàm `set_mode` sẽ kiểm tra `mode` và gọi hàm `raw_set_mode`. Hàm `raw_set_mode` sẽ gọi hàm `set_mode` cho video card tương ứng được chọn ví dụ `card->set_mode(struct mode_info*)`. Chúng ta có thể truy cập các hàm này thông qua cấu trúc `card_info`. Mỗi video mỗi định nghĩa một biến cấu trúc này và giá trị tương ứng với video mode ( ví dụ `vga` thì hàm tương ứng là `video_vga.set_mode`. Trong ví dụ về biến kiểu cấu trúc `card_info` cho `vga` ở trên). `video_vga.set_mode` là hàm `vga_set_mode`, hàm này sẽ kiểm tra vga mode và gọi hàm tương ứng function:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

Mỗi hàm setup video mode, đơn giản bên trong gọi ngắt `0x10` với giá trị tương ứng được đặt ở thanh ghi `AH`.

Sau khi thiết lập video mode, chúng ta sẽ gán vào `boot_params.hdr.vid_mode`.

Tiếp theo, hàm `vesa_store_edid` được gọi. Hàm này đơn giản là lưu thông tin [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) để kernel sử dụng. Sau bước này, hàm `store_mode_params` được gọi một lần nữa. Cuối cùng, nếu giá trị `do_restore` được set, màn hình sẽ trở về trạng thái như lúc trước.

Đến đây, chúng ta đã xong phần thiết lập video mode, giờ là lúc chuyển sang protected mode.

Chuẩn bị cuối cùng trước khi chuyển sang protected mode
--------------------------------------------------------------------------------

Bạn có thể thấy hàm chuẩn bị cuối cùng này - đó là `go_to_protected_mode` - nằm trong [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L184). Trong comment có nói: ` Làm những việc cuối cùng và kích hoạt protected mode`, vì thế chúng ta cùng xem những thứ cuối cùng này và chuyển sang protected mode.

Hàm `go_to_protected_mode` được định nghĩa trong [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c#L104). Nó chứa một vài hàm thực hiện các chuẩn bị cuối cùng trước khi nhảy đến protected mode, vì thế chúng ta hãy xem nó làm gì và nó hoạt động ra sao.

Đầu tiên là lời gọi hàm `realmode_switch_hook` bên trong hàm `go_to_protected_mode`. Hàm này sẽ kích hoạt một `hook` chuyển mode trong real mode nếu nó tồn tại và cũng tắt các [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Các hooks sẽ được sử dụng nếu bootloeader chạy ở môi trường hostile environment. Bạn có thể đọc thêm về hooks ở tài liệu [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) (xem phần **ADVANCED BOOT LOADER HOOKS**).

Cái hook `realmode_switch` là địa chỉ trỏ đến một hàm ở trong chế độ 16-bit real mode, ở đó nó sẽ tắt các ngắt không che được đi (non-maskable interrupts), tức là không cho bất cứ ngắt nào xảy ra nữa. Sau khi kiểm tra có xem có hàm `realmode_switch` hay không (trong trường hợp của tôi thì không thấy có), việc tắt các ngắt không che được được thực hiện:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

Ở trên, ta thấy đầu tiên là một hàm inline gọi lệnh asm có tên `cli` để xóa các cờ ngắt (`IF`). Sau bước này, các ngắt từ bên ngoài (external interrupt) sẽ bị tắt. Rồi tắt NMI (non-maskable interrupt).

Nói một chút về ngắt, một ngắt là một tín hiệu truyền đến CPU bởi phần mềm hoặc phần cứng. Sau khi nhận tín hiệu, CPU sẽ dừng dãy lệnh, cũng như trạng thái hiện tại, và truyền điều khiển đến hàm thực hiện ngắt (hay gọi là interrupt handler). Sau khi hàm thực hiện ngắt kết thúc công việc của nó, điều khiển sẽ được chuyển đến câu lệnh mà đã xảy ra ngắt trước đó. Ngắt không che được hay Non-maskable interrupts (NMI) là các ngắt mà luôn luôn được xử lý, không phụ thuộc quyền hạn hay hạn chế nào cả. Nó không thể bị bỏ qua, thông thường được sử dụng chuyển đến CPU một tín hiệu về các lỗi phần cứng nặng hay không thể phục hồi (non-recoverable hardware errors). Chúng ta sẽ không đi sau vào chi tiết ngắt ngay bây giờ, mà sẽ để lại ở các phần tiếp theo.

Rồi, quay trở lại với code. Chúng ta thấy dòng thứ 2 ở đoạn code phía trên, đó là ghi giá trị `0x80` ( tức là bit tắt - disabled bit) vào địa chỉ `0x70` ( thanh ghi CMOS - CMOS Address register). Sau đó, gọi hàm `io_delay` . Hàm `io_delay` này thực hiện một delay nhỏ như sau :

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

Việc đẩy bất cứ giá trị nào đến địa chỉ `0x80` sẽ gây ra delay đúng 1 microsecond. Vì thế, chúng ta có thể viết bất cứ giá trị nào (ở đây, ta lấy giá trị từ thanh ghi `AL`) vào cổng `0x80`. Sau khi delay, hàm `realmode_switch_hook` coi như kết thúc xử lý và chúng ta chuyển đến hàm tiếp theo.

Hàm tiếp theo là `enable_a20`, thực hiện bật [A20 line](http://en.wikipedia.org/wiki/A20_line). Hàm này được định nghĩa ở [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/a20.c) và nó cố gắng bật cổng A20 theo nhiều cách khác nhau. Đầu tiên trong đó là hàm `a20_test_short` nó sẽ kiểm tra xem  A20 đã được bật chưa, nếu không nó sẽ gọi hàm `a20_test`:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

    while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

Đầu tiên, chúng ta đặt giá trị `0x0000` vào thanh ghi `FS, và giá trị `0xffff` vào thanh ghi `GS`. Tiếp theo, chúng ta đọc giá trị ở địa chỉ `A20_TEST_ADDR` (đó là `0x200`), lưu các giá trị này vào biến `saved` và `ctr`.

Sau đó chúng ta đưa giá trị biến `ctr` vào `fs:gs` bằng hàm `wrfs32`, tiếp đó delay 1ms, rồi đọc giá trị từ thanh ghi `GS` bằng địa chỉ `A20_TEST_ADDR+0x10`, nếu giá trị này khác 0 thì `A20 line` đã được bật rồi. Nếu A20 đang bị tắt, chúng ta sẽ cố gắng bật nó bằng những cách khác nữa, chi tiết có ở file `a20.c`. Ví dụ gọi ngắt BIOS `0x15`  với giá trị thanh ghi `AH=0x2041` etc.

Nếu hàm `enabled_a20` kết thúc mà không bật được, nó sẽ in ra lỗi và gọi hàm `die`. Bạn có thể nhớ ra đây là, hàm này nằm trong file source đầu tiên mà chúng ta đã xem - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S):

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

Sau khi bật A20 thành công, hàm `reset_coprocessor` sẽ được gọi:
 ```C
outb(0, 0xf0);
outb(0, 0xf1);
```
Hàm này sẽ xóa cái Math Coprocessor bằng cách ghi giá trị `0` vào `0xf0`, sau đó reset bằng cách ghi `0` vào `0xf1`.

Sau bước này, hàm `mask_all_interrupts` sẽ được gọi:
```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```
Hàm này sẽ che (mask) tất cả các ngắt trên PIC (Programmable Interrupt Controller) thứ 2 (secondary PIC) và PIC thứ nhất trừ IRQ2 trên PIC chính.

Sau tất cả các bước chuẩn bị ở trên, chúng ta có thể thấy các chuyển đổi sang chế độ protected mode.

Thiết lập bảng chứa các hàm xử lý ngắt (Interrupt Descriptor Table)
--------------------------------------------------------------------------------

Giờ chúng ta sẽ thực thiện thiết lập bảng chứa hàm xử lý ngắt - Interrupt Descriptor table (IDT). `setup_idt`:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

 Nó sẽ setup IDT - Interrupt Descriptor Table ( hay các hàm xử lý ngắt, etc). Ngay bây giờ, thì IDT chưa có gì hết (chúng ta sẽ thấy nó sau), chúng ta sẽ load IDT bằng lệnh asm tên là `lidtl`. `null_idt` ở đây chứa địa chỉ và kích thước của IDT, nhưng giờ nó vẫn là zero thôi. `null_idt`  là biến kiểu cấu trúc `gdt_ptr`, nó được định nghĩa ở đây:
```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

Ở đây, ta có thể thấy một biến chứa độ dài có size 16-bit (`len`) của IDT và con trỏ 32-bit (chi tiết hơn về IDT và các ngắt được thảo luận ở các phần sau). ` __attribute__((packed))` có nghĩa rằng kích thước của `gdt_ptr` sẽ được làm cho nhỏ nhất có thể. Vì thế, ta có thể tính được kích thước của biến `gdt_ptr` sẽ là 6 bytes hay 48 bits. ( Sau đó, chúng ta sẽ load biến `gdt_ptr` vào thanh ghi `GDTR`, cái mà bạn có thể nhớ ra từ bài trước có kích thước là 48-bits).

Thiết lập bảng toàn cục (Global Descriptor Table)
--------------------------------------------------------------------------------

Giờ đến thiết lập bảng toàn cục - Global Descriptor Table (GDT). Chúng ta có thể thấy hàm `setup_gdt` được sử dụng để thiết lập GDT (bạn có thể đọc thêm về nhó ở [Kernel booting process. Part 2.](linux-bootstrap-2.md#protected-mode)). Có một định nghĩa một biến mảng `boot_gdt` trong hàm này, trong đó lại chứa định nghĩa của 3 segment:

```C
	static const u64 boot_gdt[] __attribute__((aligned(16))) = {
		[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
		[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
		[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	};
```

 Tương ứng với code, data và TSS (Task State Segment). Chúng ta sẽ không sử dụng TTS - task state segment ngay bây giờ, nó được thêm vào để làm thỏa mãn Intel VT như chúng ta có thể thấy trong nội dung comment ( nếu muốn bạn có thể tìm thấy đoạn commit của nó tại - [here](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)). Hãy xem `boot_gdt` như thế nào. Đầu tiên nhất ta thấy `__attribute__((aligned(16)))`. Có nghĩa rằng hàm này sẽ được align theo 16 bytes. Cùng xem một ví dụ như sau:
```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

Thông thường, thì một cấu trúc chỉ chứa một trường kiểu `int` như trên phải có kích thước 4 bytes, nhưng ở đây cấu trúc được thêm `aligned` như trên có kích thước 16 bytes:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

 Ở đây, `GDT_ENTRY_BOOT_CS` có index là 2, `GDT_ENTRY_BOOT_DS` thì bằng `GDT_ENTRY_BOOT_CS + 1`, etc. Nó bắt đầu từ 2, là vì phần tử đầu tiên bắt buộc phải là null (index - 0) và phần tử thứ 2 thì không được sử dụng (index - 1).

`GDT_ENTRY` là một macro, nó lấy giá trị các cờ, base rồi limit sau đó tạo phần tử cho GDT - GDT entry. Ví dụ ở đoạn tạo code segment entry. `GDT_ENTRY` sẽ lấy các giá trị sau:

* base  - 0
* limit - 0xfffff
* flags - 0xc09b

Giá trị này có nghĩa là gì? Địa chỉ segment bằng 0, giới hạn (hay kích thước segment) là `0xffff` (tức 1 MB). Các giá trị cờ thì sao?. Ở đây, nó có giá trị `0xc09b`, hay sẽ giá trị như sau:

```
1100 0000 1001 1011
```

ở dạng binary. Cùng nhau xem mỗi bit này có ý nghĩa gì. Chúng ta sẽ đi từ trái sang phải:

* 1    - (G) granularity bit
* 1    - (D) Bằng 0 tức là segment 16-bit; 1 tức là 32-bit
* 0    - (L) chạy trong 64 bit mode nếu là 1
* 0    - (AVL) sử dụng bởi phần mềm hệ thống 
* 0000 - 4 bit dành cho độ dài (19:16 bits) trong descriptor tương ứng 
* 1    - (P) có hay không segment trong bộ nhớ 
* 00   - (DPL) - privilege level, 0 là mức cao nhất 
* 1    - (S) code hoặc data segment, không phải system segment
* 101  - loại segment execute/read/
* 1    - accessed bit

 Bạn có thể đọc thêm về mỗi bit ở [post](linux-bootstrap-2.md) trước hoặc trong [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html).

Sau đó, độ dài của GDT được lấy bằng câu lệnh sau:

```C
gdt.len = sizeof(boot_gdt)-1;
```

Bằng kích thước của `boot_gdt` trừ đi 1 (giá trị địa chỉ hợp lệ cuối cùng trong GDT).

Con trỏ đến GDT được lấy như sau:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

Ở đây, ta thấy, đơn giản là lấy địa chỉ của `boot_gdt` và thêm nó vào địa chỉ của the data segment sau khi được dịch trái (left-shifted) 4 bits (nhớ là chúng ta vẫn đang ở trong real mode nha).

Cuối cùng, chúng ta thực hiện lệnh asm tên `lgdtl` để load GDT vào thanh ghi GDTR:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

Chuyển thực sự sang protected mode
--------------------------------------------------------------------------------

Đây là đoạn cuối của hàm `go_to_protected_mode`. Chúng ta đã thực hiện IDT, GDT, tắt các ngắt và giờ có thể chuyển CPU sang protected mode. Bước cuối cùng là gọi hàm `protected_mode_jump` với 2 tham số:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

Hàm này được định nghĩa ở [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S#L26). It takes two parameters:

* address of protected mode entry point 0 - địa chỉ của entry point trong chế độ protected mode
* address of `boot_params` - địa chỉ tham số boot nhân

Cùng xem bên trong hàm `protected_mode_jump`. Như đã viết ở trên, bạn có thể thấy nó ở trong file `arch/x86/boot/pmjump.S`. Tham số đầu tiên được đặt trong thanh ghi `eax` register, tham số thư 2 được đặt ở `edx`.

Đầu tiên nhất, chúng ta đặt địa chỉ của `boot_params` vào thanh ghi `esi` register và địa chỉ thanh ghi chứa code segment `cs` (0x1000) vào `bx`. Sau bước này, dịch `bx` 4 bits và cộng địa chỉ của vị trí có nhãn là `2` vào nó ( chúng ta sẽ có địa chỉ  `2` trong thanh ghi `bx` sau bước này) rồi nhảy đến địa chỉ có nhãn `1`. Sau đó, chúng ta đặt data segment và task state segment tương ứng vào thanh ghi `cs` và `di`:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

Bạn đã đọc ở trên khi `GDT_ENTRY_BOOT_CS` có index là 2, và mỗi GDT entry có kích thước là 8 byte, vì thế `CS` sẽ là `2 * 8 = 16`, `__BOOT_DS` sẽ là 24 etc.

Tiếp theo, chúng ta sẽ set bit `PE` (Protection Enable) trong thanh ghi điều khiển `CR0`:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

và thực hiện một `bước nhảy dài` sang protected mode:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

trong đó,
* `0x66` tiền tố của của hạng kích thước (operand-size prefix) cho phép chúng ta trộn code 16-bit và 32-bit,
* `0xea` - là lệnh nhảy,
* `in_pm32` là offset cho segment 
* `__BOOT_CS` là code segment.

 Sau đoạn trên, chúng ta sẽ nằm trong protected mode:

```assembly
.code32
.section ".text32","ax"
```

Hay xem những bước đầu tiên trong protected mode. Đầu tiên nhất, chúng ta phải thiết lập data segment bằng:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

Nếu bạn để ý, bạn có thể nhớ ra rằng chúng ta đã lưu `$__BOOT_DS` trong thanh ghi `cx`. Giờ chúng ta sẽ điền giá trị đó cho tất cả các thanh ghi segment bên cạnh `cs` (`cs` thì chứa `__BOOT_CS` rồi). Tiếp theo, chúng ta phải gán về ZERO cho các thanh ghi mục đích chung - general purpose registers cạnh `eax` bằng đoạn mã:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

 Cuối cùng nhảy đến điểm đầu vào 32-bit (32-bit entry point):

```
jmpl	*%eax
```

Nhớ rằng `eax` chứa địa chỉ của điểm đầu vào 32-bit (vì chúng ta đã truyền nó thông qua tham số đầu tiên của hàm `protected_mode_jump`).

Tất cả chỉ có vậy. Chúng ta đã vào trong protected mode và đang ở điểm đầu vào (entry point) của nó. Chúng ta sẽ thấy chuyện gì xảy ra tiếp theo trong phần tiếp theo của Part này.

Kết luận
--------------------------------------------------------------------------------

 Đây là đoạn cuối của phần ban trong loạt bài liên quan đến bên trong linux kernel. Trong phần tiếp theo, chúng ta sẽ xem sét các bước đầu tiên trong protected mode và cả việc chuyển sang [long mode](http://en.wikipedia.org/wiki/Long_mode) nữa.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes, please send me a PR with corrections at [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)

