 Lời gọi hệ thống (System calls) trong nhân Linux. Chương 2.
================================================================================

 Làm thế nào để nhân Linux handle một system call 
--------------------------------------------------------------------------------

 Trong phần đầu tiên [part](http://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-1.html) tức là phần đầu tiên trong chương nói về các khái niệm lời gọi hệ thống [system call](https://en.wikipedia.org/wiki/System_call) trong nhân Linux.
　Trong phần trước, chúng ta đã tìm hiểu lời gọi hệ thống (system call) là cái gì trong nhân Linux, cũng như trong các hệ điều hành nói chung. Chương này sẽ giới thiệu từ góc độ của ứng dụng (user-space), một phần của lời gọi [write](http://man7.org/linux/man-pages/man2/write.2.html) cũng đã được đem ra thao luận. Trong phần này, chúng ta tiếp tục nói về các lời gọi đó, vẫn là từ lý thuyết rồi mới vào code trong nhân Linux.

 Một ứng dụng thường sẽ không tạo trực tiếp một lời gọi hệ thống. Chúng ta thường không viết `Hello world!` giống như thế này:

```C
int main(int argc, char **argv)
{
	...
	...
	...
	sys_write(fd1, buf, strlen(buf));
	...
	...
}
```

 Chúng ta sẽ sử dụng một thứ đơn giản hơn, đó là thư viện chuẩn C ([C standard library](https://en.wikipedia.org/wiki/GNU_C_Library)) và trông nó sẽ như thế này:

```C
#include <unistd.h>

int main(int argc, char **argv)
{
	...
	...
	...
	write(fd1, buf, strlen(buf));
	...
	...
}
```

 Dù sao thì, hàm `write` ở trên không trực tiếp là một lời gọi hệ thống (system call) và không phải là một hàm từ nhân. Một ứng dụng phải điền giá trị cho các thanh ghi mục đích chung (general purpose registers) chính xác cũng như phải đúng thứ tự sau đó thực hiện lệnh ASM `syscall` để thực hiện một lời gọi hệ thống thực sự. Trong chương này, chúng ta sẽ xem chuyện gì xảy ra trong nhân Linux khi process gặp lệnh ASM `syscall`.

Khởi tạo bảng các lời gọi hệ thống (system calls table)
--------------------------------------------------------------------------------

 Từ chương trước, chúng ta đã biết rằng khái niệm lời gọi hệ thống rất giống với một ngắt (interrupt). Hơn thế, cá lời gọi hệ thống còn được thực thi giống như các ngắt mềm (software interrupt). Vì thế, khi processor gặp lệnh ÁM `syscall` từ phía ứng dụng, câu lệnh này sẽ gây ra một ngoại lệ (exception) làm Process chuyển đến hàm xử ngắt tương ứng (exception handler). Cũng như chúng ta đã biết, tất cả hàm xử lý ngoại lệ ( hay nói cách khác, các hàm trong kernel [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) mà thực hiện thông qua các ngoại lệ) được đặt trong code của nhân. Nhưng làm thế nào nhân Linux tìm được địa chỉ của các hàm xử lý lời gọi hệ thống (system call handler) cho lời gọi tương ứng? Nhân Linux chứa một bảng đặc biệt được gọi là `system call table`. Bảng lời gọi hệ thống này được biểu diễn bằng một mảng `sys_call_table` trong nhân Linux được định nghĩa trong file source code [arch/x86/entry/syscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscall_64.c). Cùng nhau xem thực thi của nó nào:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
    #include <asm/syscalls_64.h>
};
```

Như chúng ta thấy, bảng `sys_call_table` là một mảng có kích thước `__NR_syscall_max + 1`, trong đó macro `__NR_syscall_max` biểu diễn số lượng lớn nhất lời gọi hệ thống (system calls) dành cho kiến trúc [architecture](https://en.wikipedia.org/wiki/List_of_CPU_architectures). Cuốn sách này xoay quanh kiến trúc [x86_64](https://en.wikipedia.org/wiki/X86-64), trong trường hợp của chúng ta giá trị `__NR_syscall_max` là `322` và đây là giá trị tại thời điểm viết cuốn sách này ( Phiên bản hiện tại của nhân Linux là `4.2.0-rc8+`). Bạn có thể thấy giá trị này trong macro được sinh bởi [Kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt) trong quá trình build nhân - include/generated/asm-offsets.h`:

```C
#define __NR_syscall_max 322
```

 Cũng có một số lượng các lời gọi hệ thống như thế trong [arch/x86/entry/syscalls/syscall_64.tbl](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L331) trên hệ thống `x86_64`. Có 2 thứ quan trọng ở đây;  kiểu của mang biểu diễn `sys_call_table`, và giá trị khởi tạo của các phần tử trong mảng này. Đầu tiên là kiểu. Định nghĩa `sys_call_ptr_t` biểu diễn con trỏ trỏ đến một bảng lời gọi hệ thống. Nó được định nghĩa bằng [typedef](https://en.wikipedia.org/wiki/Typedef) một con trỏ hàm, với nguyên hẫu như bên dưới, không giá trị trả về, không tham số:

```C
typedef void (*sys_call_ptr_t)(void);
```

 Thứ hai, đó là giá trị khởi tạo của mảng `sys_call_table`. Như bạn thấy trong đoạn code phía trên, tất cả các phần tử mảng chúng ta đã nói đến chứa hàm xử lý lời gọi (system call handlers) trỏ đến giá trị tên `sys_ni_syscall`. Giá trị `sys_ni_syscall` là hàm biểu diễn các system call chưa được thực thi (not-implemented system calls).  Ban đầu, tất cả các phần tử của mảng `sys_call_table` cùng trỏ đến hàm xử lý rỗng hay chưa được thực thi của lời gọi hệ thống (not-implemented system call). Đây là một thao tác khởi tạo đúng đắn, bởi vì chúng ta chỉ khởi tạo nơi chứa các con trỏ đến các hàm xử lý hệ thống mà thôi, nó sẽ được điền thích hợp sau đó. Thực thi của hàm `sys_ni_syscall` khá dễ dàng, nó đơn giản trả về giá trị lỗi [-errno](http://man7.org/linux/man-pages/man3/errno.3.html) or `-ENOSYS` trong trường hợp của chúng ta như bên dưới đây:

```C
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
```

 Giá trị `-ENOSYS` được giải thích như sau:

```
ENOSYS          Function not implemented (POSIX.1)
```

 Có một chú ý ở 3 chấm `...` khi khởi tạo mảng `sys_call_table`. Bạn có thể làm điều này với tính năng mở rộng trình biên dịch trong [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) được gọi là - [Designated Initializers](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html). Phần mở rộng này cho phép khởi tạo nhiều phần tử theo thứ tự không cố định (non-fixed order). Như bạn thấy đó, chúng ta đã thêm file header `asm/syscalls_64.h` vào cuối của mảng. File header được sinh bởi một script đặc biệt, có đường dẫn [arch/x86/entry/syscalls/syscalltbl.sh](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscalltbl.sh) sẽ sinh file header cho chúng ta từ file bảng lời gọi hệ thống [syscall table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl). File header này `asm/syscalls_64.h` chứa các định nghĩa khác nhau kiểu như sau:

```C
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
__SYSCALL_COMMON(2, sys_open, sys_open)
__SYSCALL_COMMON(3, sys_close, sys_close)
__SYSCALL_COMMON(5, sys_newfstat, sys_newfstat)
...
...
...
```

 Macro `__SYSCALL_COMMON` được định nghĩa ở file source [file](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscall_64.c) được mở rộng thành macro `__SYSCALL_64` cuối cùng là thành một định nghĩa hàm:

```C
#define __SYSCALL_COMMON(nr, sym, compat) __SYSCALL_64(nr, sym, compat)
#define __SYSCALL_64(nr, sym, compat) [nr] = sym,
```

 Vì thế, sau bước này, bảng lời gọi hệ thống `sys_call_table` sẽ có dạng như sau:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
	[0] = sys_read,
	[1] = sys_write,
	[2] = sys_open,
	...
	...
	...
};
```

 Sau bước này, tất cả các phần tử mà trỏ đến những lời gọi chưa được thực thi (non-implemented system calls) sẽ chứa giá trị của hàm `sys_ni_syscall`, cái mà trả về giá trị `-ENOSYS` như chúng ta đã thấy ở trên, phần tử khác sẽ trỏ đến hàm `sys_syscall_name` tương ứng.

 Ở bước này, chúng ta đã điền giá trị cho bảng hàm xử lý của lời gọi hệ thống rồi (system call table) và từ đây, nhân Linux sẽ biết hàm xử lý cho mỗi lời gọi ở đâu. Nhưng nhân Linux kernel không gọi các hàm `sys_syscall_name` ngay lập tức sau khi nó được chỉ thị để xử lý một lời gọi hệ thống từ ứng dụng phía user-space. Hãy nhớ lại trongll [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) về các ngắt và các hàm xử lý ngắt.  Khi nhân Linux chuyển điều khiển sang xử lý một ngắt, nó phải thực hiện các bước chuẩn bị như cất các thanh ghi của user-space (user space registers), rồi thì chuyển sang một stack mới, cùng rất nhiều nhiệm vụ khác nữa trước khi nó thực sự thực hiện hàm xử lý ngắt. Tình huống cũng tương tự với xử lý lời gọi hệ thống (system call handling). Việc chuẩn bị để xử lý một lời gọi hệ thống là thứ đầu tiên, nhưng trước khi nhân Linux bắt đầu thực hiện những chuẩn bị, điểm bắt đầu (the entry point) của một lời gọi hệ thống phải được khởi tạo và chỉ nhân Linux mới biết làm thế nào để thực hiện việc này. Trong đoạn sau, chúng ta sẽ nói về quá trình khởi tạo một đầu vào lời gọi hệ thống (system call entry) trong Linux kernel.

 Quá trình khởi đầu vào của lời gọi hệ thống (system call entry)
--------------------------------------------------------------------------------

 Khi một lời gọi hệ thống được thực hiện, đâu sẽ là byte đầu tiên của code mà CPU sẽ chạy? Bạn có thể đọc nó trong tài liệu hướng dẫn của Intel - [64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html):

```
SYSCALL invokes an OS system-call handler at privilege level 0.
It does so by loading RIP from the IA32_LSTAR MSR
```

 Điều này có nghĩa là, chúng ta cần đặt điểm vào của lời gọi hệ thống vào `IA32_LSTAR` [model specific register](https://en.wikipedia.org/wiki/Model-specific_register). Thao tác này được thực hiện trong quá trình khởi tạo nhân Linux rồi. Nếu bạn đã đọc phần 4 [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-4.html) của chương nói về ngắt và xử lý ngắt trong nhân Linux, bạn sẽ biết rằng nhân Linux gọi hàm `trap_init` trong quá trình khởi tạo. Hàm này được định nghĩa ở file source [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) và nó thực hiện khởi tạo các hàm xử lý ngoại lệ sau (`non-early`) như lỗi chia, lỗi [coprocessor](https://en.wikipedia.org/wiki/Coprocessor) vân vân. Ngoài việc khởi tạo các hàm xử lý ngoại lệ sau (`non-early`), hàm này cũng gọi `cpu_init` từ file source [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/blob/arch/x86/kernel/cpu/common.c) cái sẽ khởi tạo trạng thái cho mỗi CPU (`per-cpu` state), cuối cùng thực hiện hàm `syscall_init` ở cùng file đó.

 
 Hàm bên dưới đây sẽ thực hiện khởi tạo điểm vào của lời gọi hệ thống. Cùng nhau xem xét thực thi của nó. Nó không nhận tham số, nó thực hiện điền giá trị 2 thanh ghi chỉ định model (two model specific registers):

```C
wrmsrl(MSR_STAR,  ((u64)__USER32_CS)<<48  | ((u64)__KERNEL_CS)<<32);
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

 Thanh ghi model đầu tiên - `MSR_STAR` có dãy bit `63:48` dành cho địa chỉ của đoạn code user (user code segment). Những giá trị này được load vào các thanh ghi segment `CS` và `SS` cho câu lệnh `sysret` giúp nó cung cấp chức năng trả khi chuyển điều khiển từ lời gọi hệ thống sang code phía user kèm theo những giá trị lien quan đến quyền thưc thi (privilege). Cũng ở thanh ghi này `MSR_STAR` dãy bit `47:32` chứa địa chỉ code từ kernel, cái sẽ được sử dụng là base selector cho các thanh ghi segment `CS` và `SS` segment khi ứng dụng phía user-space muốn chạy một lời gọi. Trong dòng code thứ hai, chúng ta gán giá trị thanh ghi `MSR_LSTAR` bằng giá trị `entry_SYSCALL_64` chính là địa chỉ của điểm vào lời gọi hệ thống. Giá trị `entry_SYSCALL_64` được định nghĩa trong file source code asm [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) và chứa code liên quan đến việc khởi, được chạy trước khi lời hàm xử lý lời gọi hệ thống được chạy ( tôi đã viết về quá trình khởi tạo này, xin hãy đọc ở trên). Chúng ta sẽ không xem xét kĩ `entry_SYSCALL_64` ngay bây giờ, chúng ta sẽ bàn thêm ở phần sau cùa chương này.

 Sau khi thiết lập giá trị điểm đầu vào cho lời gọi hệ thống, chúng ta cần thiết lập các thanh ghi model như sau:

* `MSR_CSTAR` - target `rip` cho lời gọi ở chế độ tương thích;
* `MSR_IA32_SYSENTER_CS` - target `cs` cho câu lệnh asm `sysenter`;
* `MSR_IA32_SYSENTER_ESP` - target `esp` cho câu lệnh asm `sysenter`;
* `MSR_IA32_SYSENTER_EIP` - target `eip` cho câu lệnh asm `sysenter`.

 Giá trị của những thanh ghi chỉ định phụ thuộc vào giá trị `CONFIG_IA32_EMULATION` trong cấu hình nhân. Nếu giá trị đó được bật, nó sẽ cho phép một chương trình legacy 32-bit chạy ở môi trường sử dụng nhân 64-bit. Trường hợp đầu tiên, nếu giá trị cấu hình nhân `CONFIG_IA32_EMULATION` được bật, chúng ta sẽ gán giá trị cho các thanh ghi chỉ định model dành cho mode tương thích:

```C
wrmsrl(MSR_CSTAR, entry_SYSCALL_compat);
```

 còn với code của kernel, đẩy 0 vào stack và viết địa chỉ của `entry_SYSENTER_compat` vào con trỏ câu lệnh [instruction pointer](https://en.wikipedia.org/wiki/Program_counter):

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
```

 Trong các trường hợp khác, nếu giá trị cấu hình nhân `CONFIG_IA32_EMULATION` bị tắt, chúng ta sẽ điền địa chỉ của `ignore_sysret` vào thanh ghi `MSR_CSTAR`:

```C
wrmsrl(MSR_CSTAR, ignore_sysret);
```

 đoạn này được định nghĩa ở file source asm [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) và trả về giá trị mã lỗi `-ENOSYS` nếu có lỗi xảy ra:

```assembly
ENTRY(ignore_sysret)
	mov	$-ENOSYS, %eax
	sysret
END(ignore_sysret)
```

 Giờ bạn cần điền giá trị cho thanh ghi quy định model khác như `MSR_IA32_SYSENTER_CS`, `MSR_IA32_SYSENTER_ESP`, `MSR_IA32_SYSENTER_EIP` như chúng ta đã làm trong trường hợp giá trị cấu hình nhân `CONFIG_IA32_EMULATION` được bật. Trong trường hợp này ( tức là trường hợp khi giá trị `CONFIG_IA32_EMULATION` không được bật) chúng ta sẽ điền giá trị cho thanh ghi `MSR_IA32_SYSENTER_ESP` và `MSR_IA32_SYSENTER_EIP` bằng ZERO và điền một cái segment không hợp lệ trong [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) cho thanh ghi chỉ định model `MSR_IA32_SYSENTER_CS`:

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
```

 Bạn có thể đọc thêm về `Global Descriptor Table` trong [part](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html) 2 của chương miên tả về quá trình nhân Linux.

 Cuối hàm `syscall_init`, chúng ta sử dụng gán giá trị [flags register](https://en.wikipedia.org/wiki/FLAGS_register) bằng việc ghi giá trị  cờ đó vào thanh ghi chỉ định model `MSR_SYSCALL_MASK`:

```C
wrmsrl(MSR_SYSCALL_MASK,
	   X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
	   X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
```

 Các cờ này đã được xóa trong lúc khởi tạo rồi. Đến đây, là toàn bộ xử lý của hàm `syscall_init` từ đây, các điểm vào cho lời gọi hệ thống đã sẵn sàng làm việc rồi. Giờ chúng ta tiếp tục xem chuyện gì xảy ra khi một ứng dụng phía user thực hiện lệnh asm `syscall`.

 Các chuẩn bị trước khi hàm xử lý lời gọi hệ thống được gọi
--------------------------------------------------------------------------------

 Như tôi đã viết trước đó, trước khi hàm xử lý ngắt hay hàm xử lý lời gọi hệ thống được gọi bởi Linux kernel chúng ta có một vài chuẩn bị. Macro `idtentry` thực hiện một số chuẩn bị bắt buộc trước khi hàm xử lý ngoại lệ được gọi, macro `interrupt` cũng thực hiện các chuẩn bị cần thiết trước khi hàm xử lý ngắt được gọi, cuối cùng macro `entry_SYSCALL_64` sẽ thực hiện các chuẩn bị bắt buộc trước khi hàm xử lý lời gọi hệ thống được thực hiện.

Giá trị `entry_SYSCALL_64` được định nghĩa ở file source asm [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) được bắt đầu ở macro sau:

```assembly
SWAPGS_UNSAFE_STACK
```

 Macro này được nghĩa ở file header [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h) và được mở rộng thành lệnh asm `swapgs`: 

```C
#define SWAPGS_UNSAFE_STACK	swapgs
```

 Lệnh này sẽ hoán đổi giá trị của thanh ghi GS base với giá trị của thanh ghi chỉ đinh model `MSR_KERNEL_GS_BASE ` . Hay nói cách khác chúng ta sẽ chuyển nó đến stack của kernel. Sau khi trỏ địa chỉ cũng đến giá trị biến `rsp_scratch` [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) và thiết lập cho trỏ stack (SP - stack pointer) lên đầu stack của processor hiện tại:

```assembly
movq	%rsp, PER_CPU_VAR(rsp_scratch)
movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```

Bước tiếp theo, chúng ta đẩy stack segment và stack pointer cũ vào stack:

```assembly
pushq	$__USER_DS
pushq	PER_CPU_VAR(rsp_scratch)
```

 Sau bước này, chúng ta sẽ bật các ngắt, bởi vì các ngắt đã bị `off` khi bắt đầu vào lúc lưu lại các giá trị thanh ghi [registers](https://en.wikipedia.org/wiki/Processor_register) (ngoài `bp`, `bx` và từ `r12` đến `r15`), các cờ, giá trị `-ENOSYS` dành cho các system call không được implement, cuối cùng là thanh ghi segment code trên stack:

```assembly
ENABLE_INTERRUPTS(CLBR_NONE)

pushq	%r11
pushq	$__USER_CS
pushq	%rcx
pushq	%rax
pushq	%rdi
pushq	%rsi
pushq	%rdx
pushq	%rcx
pushq	$-ENOSYS
pushq	%r8
pushq	%r9
pushq	%r10
pushq	%r11
sub	$(6*8), %rsp
```

 Khi một system call xảy ra từ phía ứng dụng user, các thanh ghi mục đích chung sẽ được sử dụng như sau:

* `rax` - chứa system call number; 
* `rcx` - chứa địa chỉ trả về phía user space;
* `r11` - chứa cơ thanh ghi (register flags);
* `rdi` - chứa tham số đầu tiên (first argument) của system call handler;
* `rsi` - chứa tham số thứ hai của system call handler;
* `rdx` - chứa tham số thứ ba của system call handler;
* `r10` - chứa tham số thứ tư của system call handler;
* `r8`  - chứa tham số thứ năm của system call handler;
* `r9`  - chứa tham số thứ sau của system call handler;

Các thanh ghi mục đích chung khác (as `rbp`, `rbx` and from `r12` to `r15`) được gọi là callee-preserved in [C ABI](http://www.x86-64.org/documentation/abi.pdf)). Vì thế, chúng ta đặt các cờ thanh ghi lên đỉnh stack, tiếp đó đến segment của user code, rồi địa chỉ trả về user space, chỉ số của lời gọi hệ thống, tham số đầu tiên, code dump với những lời gọi không được implement, rồi các tham số khác nữa.

 Bước tiếp theo, chúng ta sẽ kiểm trả giá trị `_TIF_WORK_SYSCALL_ENTRY` trong `thread_info` hiện tại:

```assembly
testl	$_TIF_WORK_SYSCALL_ENTRY, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
jnz	tracesys
```

 Macro `_TIF_WORK_SYSCALL_ENTRY` được định nghĩa trong file header[arch/x86/include/asm/thread_info.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/thread_info.h), nó cung cấp một tập thông tin các cờ thông tin thread, cái liên quan đến việc trace lời gọi hệ thống:

```C
#define _TIF_WORK_SYSCALL_ENTRY \
    (_TIF_SYSCALL_TRACE | _TIF_SYSCALL_EMU | _TIF_SYSCALL_AUDIT |   \
    _TIF_SECCOMP | _TIF_SINGLESTEP | _TIF_SYSCALL_TRACEPOINT |     \
    _TIF_NOHZ)
```

 Chúng ta sẽ không xem những thứ liên quan đến debugging/tracing trong chương này, chúng ta sẽ xem xét chúng trong một chương khác, cái sẽ tập trung vào các kĩ thuật debugging và tracing techniques trong nhân Linux. Sau nhãn `tracesys`, ta sẽ thấy nhãn `entry_SYSCALL_64_fastpath`. Với nhãn `entry_SYSCALL_64_fastpath` này, chúng ta sẽ kiểm tra macro `__SYSCALL_MASK`, giá trị mà đã được định nghĩa trong file header [arch/x86/include/asm/unistd.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/unistd.h) và

```C
# ifdef CONFIG_X86_X32_ABI
#  define __SYSCALL_MASK (~(__X32_SYSCALL_BIT))
# else
#  define __SYSCALL_MASK (~0)
# endif
```

 Trong đó, macro `__X32_SYSCALL_BIT` là 

```C
#define __X32_SYSCALL_BIT	0x40000000
```

 Như bạn thấy `__SYSCALL_MASK` phụ thuộc vào giá trị cấu hình nhân `CONFIG_X86_X32_ABI`, nó biểu diễn một mask 32-bit cho [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) trong nhân 64-bit.

 Vì thế, chúng ta kiểm tra giá trị `__SYSCALL_MASK` nếu `CONFIG_X86_X32_ABI` is disabled we compare the value of the `rax` register to the maximum syscall number (`__NR_syscall_max`), alternatively if the `CONFIG_X86_X32_ABI` is enabled we mask the `eax` register with the `__X32_SYSCALL_BIT` and do the same comparison: 

```assembly
#if __SYSCALL_MASK == ~0
	cmpq	$__NR_syscall_max, %rax
#else
	andl	$__SYSCALL_MASK, %eax
	cmpl	$__NR_syscall_max, %eax
#endif
```

After this we check the result of the last comparison with the `ja` instruction that executes if `CF` and `ZF` flags are zero:

```assembly
ja	1f
```

and if we have the correct system call for this, we move the fourth argument from the `r10` to the `rcx` to keep [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf) compliant and execute the `call` instruction with the address of a system call handler: 

```assembly
movq	%r10, %rcx
call	*sys_call_table(, %rax, 8)
```

Note, the `sys_call_table` is an array that we saw above in this part. As we already know the `rax` general purpose register contains the number of a system call and each element of the `sys_call_table` is 8-bytes. So we are using `*sys_call_table(, %rax, 8)` this notation to find the correct offset in the `sys_call_table` array for the given system call handler.

That's all. We did all the required preparations and the system call handler was called for the given interrupt handler, for example `sys_read`, `sys_write` or other system call handler that is defined with the `SYSCALL_DEFINE[N]` macro in the Linux kernel code.

Exit from a system call
--------------------------------------------------------------------------------

After a system call handler finishes its work, we will return back to the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S), right after where we have called the system call handler:

```assembly
call	*sys_call_table(, %rax, 8)
```

The next step after we've returned from a system call handler is to put the return value of a system handler on to the stack. We know that a system call returns the result to the user program in the general purpose `rax` register, so we are moving its value on to the stack after the system call handler has finished its work:

```C
movq	%rax, RAX(%rsp)
```

on the `RAX` place.

After this we can see the call of the `LOCKDEP_SYS_EXIT` macro from the [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h):

```assembly
LOCKDEP_SYS_EXIT
```

The implementation of this macro depends on the `CONFIG_DEBUG_LOCK_ALLOC` kernel configuration option that allows us to debug locks on exit from a system call. And again, we will not consider it in this chapter, but will return to it in a separate one. In the end of the `entry_SYSCALL_64` function we restore all general purpose registers besides `rcx` and `r11`, because the `rcx` register must contain the return address to the application that called system call and the `r11` register contains the old [flags register](https://en.wikipedia.org/wiki/FLAGS_register). After all general purpose registers are restored, we fill `rcx` with the return address, `r11` register with the flags and `rsp` with the old stack pointer:

```assembly
RESTORE_C_REGS_EXCEPT_RCX_R11

movq	RIP(%rsp), %rcx
movq	EFLAGS(%rsp), %r11
movq	RSP(%rsp), %rsp

USERGS_SYSRET64
```

In the end we just call the `USERGS_SYSRET64` macro that expands to the call of the `swapgs` instruction which exchanges again the user `GS` and kernel `GS` and the `sysretq` instruction which executes on exit from a system call handler:

```C
#define USERGS_SYSRET64				\
	swapgs;	           				\
	sysretq;
```

Now we know what occurs when a user application calls a system call. The full path of this process is as follows:

* User application contains code that fills general purpose register with the values (system call number and arguments of this system call);
* Processor switches from the user mode to kernel mode and starts execution of the system call entry - `entry_SYSCALL_64`; 
* `entry_SYSCALL_64` switches to the kernel stack and saves some general purpose registers, old stack and code segment, flags and etc... on the stack;
* `entry_SYSCALL_64` checks the system call number in the `rax` register, searches a system call handler in the `sys_call_table` and calls it, if the number of a system call is correct;
* If a system call is not correct, jump on exit from system call;
* After a system call handler will finish its work, restore general purpose registers, old stack, flags and return address and exit from the `entry_SYSCALL_64` with the `sysretq` instruction. 

That's all.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about the system calls concept in the Linux kernel. In the previous [part](http://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-1.html) we saw theory about this concept from the user application view. In this part we continued to dive into the stuff which is related to the system call concept and saw what the Linux kernel does when a system call occurs.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [write](http://man7.org/linux/man-pages/man2/write.2.html)
* [C standard library](https://en.wikipedia.org/wiki/GNU_C_Library)
* [list of cpu architectures](https://en.wikipedia.org/wiki/List_of_CPU_architectures)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
* [typedef](https://en.wikipedia.org/wiki/Typedef)
* [errno](http://man7.org/linux/man-pages/man3/errno.3.html)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [model specific register](https://en.wikipedia.org/wiki/Model-specific_register)
* [intel 2b manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [coprocessor](https://en.wikipedia.org/wiki/Coprocessor)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [flags register](https://en.wikipedia.org/wiki/FLAGS_register)
* [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [general purpose registers](https://en.wikipedia.org/wiki/Processor_register)
* [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)
* [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf)
* [previous chapter](http://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-1.html)
