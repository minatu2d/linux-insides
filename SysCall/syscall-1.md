 Các lời gọi hệ thống (System calls) trong nhân Linux (Linux kernel). Part 1 
================================================================================

Giới thiệu
--------------------------------------------------------------------------------

 Đây là bài đâu tiên của một chương mới trong cuốn sách [linux-insides](http://0xax.gitbooks.io/linux-insides/content/), như bạn có thể thấy từ tựa đề, chương này sẽ nói về khái niệm [System call](https://en.wikipedia.org/wiki/System_call) trong Linux kernel. Lựa chọn chủ đề sẽ nói trong chương này cũng không có gì bất ngời. Như đã nói ở chương trước [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chúng ta đã bàn về ngắt (interrupts) và việc xử lý ngắt (interrupt handling). Các khái niệm liên quan lời gọi hệ thống (system calls) rất giống với các ngắt đã nói. Đó là vì hầu hết lời gọi hệ thống được thực hiện (implement) giống như ngắt mềm (software interrupts). Chúng ta sẽ thấy có rất nhiều khía cạnh khác nhau liên quan đến khái niệm lời gọi hệ thống (system call). Ví dụ, chúng ta sẽ tìm hiểu xem cái gì xảy ra khi một lời gọi hệ thống (system call) được phía user-space yêu cầu thực hiện. Chúng ta cũng sẽ xem thử thực thi của một vài lời gọi hệ thống (a couple system call handlers)  trong nhân Linux, khái niệm [VDSO](https://en.wikipedia.org/wiki/VDSO), [vsyscall](https://lwn.net/Articles/446528/) cùng rất nhiều thứ khác.

 Trước khi đi sâu vào xem các thực thi của lời gọi hệ thống trong Linux (Linux system call implementation), chúng ta nên tìm hiểu qua về lý thuyết (theory) về các lời gọi hệ thống (system calls). Nào hãy xem nó ở phần sau đây.

 Lời gọi hệ thống (System call). Nó là cái gì?
--------------------------------------------------------------------------------

 Một lời gọi hệ thống (A system call) đơn giản là một yêu cầu từ userspace (userspace request) đến dịch vụ của nhân (kernel service). Vâng, như chúng ta đã biết nhân hệ điều hành (operating system kernel) cung cấp rất nhiều dịch vụ (service). Đó là khi chương trình của bạn muốn ghi hoặc đọc dữ liệu từ một file file,  khi bắt đầu lắng nghe kết nối trên một [socket](https://en.wikipedia.org/wiki/Network_socket), khi xóa hoặc tạo thư mục, hoặc thâm chí khi kết thúc chương trình, thì chương trình của bạn đều chắc chắn đang sử lời gọi hệ thống (system call). Hay nói một cách khác, một lời gọi chíng là một hàm của ngôn ngữ [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) trong kernel space khi chương trình phía user space gọi để yêu cầu một vài thứ nó muốn.

 Nhân Linux (Linux kernel) cung cấp một tập các hàm kiểu này và mỗi kiến trúc phần cứng cũng có một tập riêng của nó. Ví dụ: Kiến trúc [x86_64](https://en.wikipedia.org/wiki/X86-64) cung cấp [322](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) lời gọi (system calls) và kiến trúc [x86](https://en.wikipedia.org/wiki/X86) cung cấp [358](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_32.tbl) lời gọi (system calls) khác. Rồi, đến đây ta đã hiểu một lời gọi hệ thống (system call) đơn giản là một hàm (function). Giờ, hãy cùng xem một ví dụ đơn giản như `Hello world` được viết trong ngôn ngữ lập trình assembly:

```assembly
.data

msg:
    .ascii "Hello, world!\n"
    len = . - msg

.text
    .global _start

_start:
	movq  $1, %rax
    movq  $1, %rdi
    movq  $msg, %rsi
    movq  $len, %rdx
    syscall

    movq  $60, %rax
    xorq  %rdi, %rdi
    syscall
```

 Chúng ta có thê biên dịch đoạn code ở trên bằng câu lệnh dưới đây:

```
$ gcc -c test.S
$ ld -o test test.o
```

 và chạy nói như thế này:

```
./test
Hello, world!
```

Ok, chúng ta thấy gì ở đây? Một chương trình hiển thị dòng chữ `Hello world` viết bằng assembly cho Linux chạy trên kiến trúc phần cứng `x86_64`. Trong đoạn code ở trên, chúng ta sẽ để ý 2 section sau:

* `.data`
* `.text`

Section đầu tiên - `.data` lưu dữ liệu khởi tạo cho chương trình của chúng ta ( ở đây, đó là chuỗi kí tự `Hello world` và độ dài của nó). Section thứ 2 - `.text` chứa code cho chương trình của chúng ta. Chúng ta tiếp tục chia code thành 2 phần: phần đầu tiên là đoạn nằm trước lệnh `syscall` đầu tiên và phần thứ 2 nằm giữa 2 câu lệnh  `syscall`. Đầu tiên, ta tự hỏi rằng cái lệnh assembly `syscall` làm cái quái gì trong code của chúng ta thế? Không nơi nào khác rõ hơn chính giải thích của Intel, tại [64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html):
**TODO : Lack of knowledge in Intel CPU, translate later**
```
SYSCALL invokes an OS system-call handler at privilege level 0. It does so by
loading RIP from the IA32_LSTAR MSR (after saving the address of the instruction
following SYSCALL into RCX). (The WRMSR instruction ensures that the
IA32_LSTAR MSR always contain a canonical address.)
...
...
...
SYSCALL loads the CS and SS selectors with values derived from bits 47:32 of the
IA32_STAR MSR. However, the CS and SS descriptor caches are not loaded from the
descriptors (in GDT or LDT) referenced by those selectors.

Instead, the descriptor caches are loaded with fixed values. It is the respon-
sibility of OS software to ensure that the descriptors (in GDT or LDT) referenced
by those selector values correspond to the fixed values loaded into the descriptor
caches; the SYSCALL instruction does not ensure this correspondence.
```

 và chúng ta sẽ thực hiện khởi tạo các `syscalls` thông qua việc ghi giá trị `entry_SYSCALL_64` đã được định nghĩa trong file assembly [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) và đưa câu lệnh `SYSCALL` vào `IA32_STAR` [Model specific register](https://en.wikipedia.org/wiki/Model-specific_register):

```C
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

trong file mã nguồn [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c).

 Và thế là, câu lệnh `syscall` sẽ yêu cầu một hàm điều khiển (handler) cho lời gọi hệ thống chúng ta yêu cầu (given system call). Nhưng làm thế nào biết cái hàm điều khiển nào nên được gọi đây (which handler to call)? Sự thực là, nó sẽ lấy thông tin từ các thanh ghi mục đích chung (general purpose) [registers](https://en.wikipedia.org/wiki/Processor_register). Bạn có thể thấy trong bảng các lời hệ thống (system call [table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)), mỗi lời gọi được chỉ định bởi một số duy nhất. Trong ví dụ của chúng ta, lời gọi đầu tiên - `write` thực hiện ghi dữ liệu vào một file được chỉ định. Hãy thử xem bảng lời gọi hệ thống (system call table) và thử tìm lời gọi `write` thử xem nào. Như ta sẽ thấy, lời gọi [write](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L10) được gán một giá trị - `1`. Chúng ta gán con số tương ứng với lời gọi thông qua thanh ghi `rax` như đã thấy trên ví dụ. Các thanh ghi mục đích chung tiếp theo: `%rdi`, `%rsi` and `%rdx` được sử dụng để lưu tham số của lời gọi `write`. Trong trường hợp của chúng ta, các tham số đầu tiên đó là [file descriptor](https://en.wikipedia.org/wiki/File_descriptor) (`1` tương ứng là [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29) trong trường hợp này), tham số thứ 2 là con trỏ chỉ đến chuỗi kí tự (chính là `Hello World`), và tham số thứ 3 là độ dài chuỗi kí tự. Vâng, bạn không nghe nhầm đầu. Những tham số này dành cho một lời gọi thôi. Như đã nói ở phần trên, một lời gọi đơn giả là một hàm `C` trong kernel space. Trong trường hợp của chúng ta, lời gọi đó là `write`. Định nghĩa của lời gọi này được viết ở file mã nguồn [fs/read_write.c](https://github.com/torvalds/linux/blob/master/fs/read_write.c) và trông nó như thế này:

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	...
	...
	...
}
```

 Hay diễn đạt đơn giản hơn, nó như thế này:

```C
ssize_t write(int fd, const void *buf, size_t nbytes);
```

 Đừng quá lo lắng về macro `SYSCALL_DEFINE3` kia ngay lúc này, chúng ta sẽ quay lại với sớm thôi.

 Phần code thứ 2 của chúng ta cũng tương tự thôi, nhưng nó cho một lời gọi khác. Đó là lời gọi [exit](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L69). Lời gọi này chỉ nhận một tham số thôi, đó là:

* Return value
 
 và đảm nhiệm việc thoát chương trình. Chúng ta có thể sử dụng công cụ [strace](https://en.wikipedia.org/wiki/Strace) để xem các lời gọi trong chương trình trên, cách thực hiện như sau:

```
$ strace test
execve("./test", ["./test"], [/* 62 vars */]) = 0
write(1, "Hello, world!\n", 14Hello, world!
)         = 14
_exit(0)                                = ?

+++ exited with 0 +++
```

 Dòng đầu tiên trong kết quả từ `strace`, chúng ta có thể thấy lời gọi [execve](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L68) để thực hiện chương trình chúng ta truyền vào, lời gọi thứ 2, 3 thì giống như ta đã phân tích ở trên, chính là: `write` and `exit`. Nhớ rằng, trong ví dụ trên, chúng ta truyền tham số thông qua các thanh ghi mục đích chung (general purpose registers). Thứ tự các thanh ghi cũng không có gì đáng nói lắm. Thứ tự này được quy định trong quy ước như sau - [x86-64 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions). Quy ước này và những quy ước khác của kiến trúc `x86_64` được miêu tả kĩ hơn trong tài liệu đặc biệt có tên - [System V Application Binary Interface. PDF](http://www.x86-64.org/documentation/abi.pdf). Về cơ bản, các tham số được đặt hoặc trong thanh ghi hoặc được đẩy vào stack. Thứ tự đúng là:

* `rdi`;
* `rsi`;
* `rdx`;
* `rcx`;
* `r8`;
* `r9`.

cho một hàm có 6 tham số. 
 Nếu hàm đó có nhiều hơn 6 tham số, các tham số sau sẽ được đưa đẩy vào stack.

 Chúng ta thường không sử dụng trực tiếp lời gọi hệ thống trong code chương trình, nhưng chương trình của chúng ta sẽ phải sử dụng chúng khi muốn in gì đó ra, kiểm tra truy cập đến một file hoặc hoặc đơn giản là đọc/ghi vào nó.

 Ví dụ:

```C
#include <stdio.h>

int main(int argc, char **argv)
{
   FILE *fp;
   char buff[255];

   fp = fopen("test.txt", "r");
   fgets(buff, 255, fp);
   printf("%s\n", buff);
   fclose(fp);

   return 0;
}
```

 Không hề có lời gọi hệ thống nào là `fopen`, `fgets`, `printf` and `fclose` trong nhân Linux, các hàm `open`, `read` `write` và `close` được sử dụng. Tôi nghĩ bạn sẽ biết 4 hàm `fopen`, `fgets`, `printf` và `fclose` này, nó chính là các hàm được định nghĩa trong `C` [standard library](https://en.wikipedia.org/wiki/GNU_C_Library). Thực sự thì, những hàm này thực hiện việc wrapper (bọc) các lời gọi hệ thống khác (tức là gọi lời gọi hệ thống bên trong thực thi của hàm đó và cung cấp một giao diện đơn giản hơn ra bên ngoài). Chúng ta tốt nhất đừng gọi trực tiếp các lời gọi hệ thống trong code chương trình, thay vào đó, hãy sử dụng các hàm [wrapper](https://en.wikipedia.org/wiki/Wrapper_function) từ thư viện chuẩn (standard library). Lý do chính của việc này là đơn giản là: một lời gọi hệ thống được thực hiện nhanh, rất nhanh. Để thực hiện nhanh, nó phải nhỏ. Thư viện chuẩn (standard library) chịu tránh nhiệm thực hiện các lời gọi hệ thống với tập tham số được đảm bảo, rồi thực hiện các bước kiểm tra (check) trước khi thực sự gọi các lời gọi hệ thống. Hãy thực hiện chương trình của chúng ta bằng dòng lệnh sau:

```
$ gcc test.c -o test
```

 Bây giờ kiểm tra nó với một công cụ khác, có tên là [ltrace](https://en.wikipedia.org/wiki/Ltrace):

```
$ ltrace ./test
__libc_start_main([ "./test" ] <unfinished ...>
fopen("test.txt", "r")                                             = 0x602010
fgets("Hello World!\n", 255, 0x602010)                             = 0x7ffd2745e700
puts("Hello World!\n"Hello World!

)                                                                  = 14
fclose(0x602010)                                                   = 0
+++ exited (status 0) +++
```

 Câu lệnh `ltrace` hiển thị một tập các lời gọi (call) của userspace khi thực hiện chương trình. Hàm `fopen` thực hiện mở một file text, hàm `fgets` đọc một đoạn nội dung vào buffer `buf`, hàm `puts` thực hiện in nội dung đọc được ra `stdout` và hàm `fclose` đóng file đã mở thông qua con trỏ file được trả về ở `fopen` trước đó. Và như tôi đã viết lúc trước, tất cả các hàm trên sẽ gọi các lời gọi hệ thống thích hợp. Ví dụ `puts` sẽ gọi lời gọi `write` bên trong, chúng ta có thể thấy điều này nếu thêm option `-S` vào câu lệnh `ltrace`:

```
write@SYS(1, "Hello World!\n\n", 14) = 14
```
 
 Vâng, lời gọi hệ thống ở khắp mọi nơi. Bất cứ chương trình nào cũng cần mở/ghi/đọc file, kết nối mạng, cấp phát bộ nhớ và rất nhiều thứ khác nữa mà chỉ có thể được cung cấp từ kernel. Hệ thống file [proc](https://en.wikipedia.org/wiki/Procfs) chứa những file đặc biệt có dạng như: `/proc/pid/systemcall`, chính ở đó nó sẽ "phơi" ra con số của lời gọi hệ thống (system call number) và các thanh ghi tham số (argument registers) khi các lời gọi đó được chạy bởi tiến trình tương ứng. Ví dụ, pid 1, đó là [systemd](https://en.wikipedia.org/wiki/Systemd) trên hệ thống của tôi:

```
$ sudo cat /proc/1/comm
systemd

$ sudo cat /proc/1/syscall
232 0x4 0x7ffdf82e11b0 0x1f 0xffffffff 0x100 0x7ffdf82e11bf 0x7ffdf82e11a0 0x7f9114681193
```

 lời gọi hệ thống - `232`, chính là lời gọi hệ thống tên[epoll_wait](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L241) được sử dụng để đợi I/O event một file descriptor [epoll](https://en.wikipedia.org/wiki/Epoll). Hoặc ví dụ trường hợp trình soạn thảo `emacs` cái tôi đang viết tài liệu này:

```
$ ps ax | grep emacs
2093 ?        Sl     2:40 emacs

$ sudo cat /proc/2093/comm
emacs

$ sudo cat /proc/2093/syscall
270 0xf 0x7fff068a5a90 0x7fff068a5b10 0x0 0x7fff068a59c0 0x7fff068a59d0 0x7fff068a59b0 0x7f777dd8813c
```

Lời gọi hệ thống có số là `270`, đó là lời gọi [sys_pselect6](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L279) cho phép `emacs` quan sát trên nhiều file descriptor.

 Giờ chúng ta đã biết một chút về lời gọi hệ thống (system call) rồi, chúng ta đã biết nó là gì và tại sao lại cần nó rồi. Giờ hãy xem chi tiết lời gọi `write` mà chương trình chúng ta đang sử dụng.

 Thực thi (implementation) của lời gọi hệ thống write
--------------------------------------------------------------------------------

 Cùng xem thực thi của lời gọi hệ thống (system call) này trực tiếp từ (source code) của nhân Linux. Như chúng ta đã biết, lời gọi hệ thống `write` được định nghĩa tại file source [fs/read_write.c](https://github.com/torvalds/linux/blob/master/fs/read_write.c) và hãy cùng xem nó:

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

 Đầu tiên, chúng ta thấy macro `SYSCALL_DEFINE3` được định nghĩa ở file header [include/linux/syscalls.h](https://github.com/torvalds/linux/blob/master/include/linux/syscalls.h), nó thay thế cho đoạn định nghĩa hàm `sys_name(...)`. Hãy cùng nhau xem macro này:

```C
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)                \
        SYSCALL_METADATA(sname, x, __VA_ARGS__)       \
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

 Như chúng thấy ở chỗ macro trên  `SYSCALL_DEFINE3` được sử dụng, nó nhận tham số `name`, biểu diễn tên của lời gọi hệ thống (system call) và kí hiệu tham số biến đổi được (variadic number of parameters). Macro đơn giản là mở rộng tiếp bằng macro `SYSCALL_DEFINEx`, nó sẽ lấy số lượng tham số được truyền vào cho lời gọi, tên `_##name` để sử dụng trong tương lai của lời gọi ( nếu muốn biết thêm vì cách nối kí hiệu kiểu `##` này bạn có thể tìm đọc tại [documentation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html) trong [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)). Giờ chúng ta cùng xem macro `SYSCALL_DEFINEx`. Macro này mở rộng thông qua 2 macro bên dưới đây:

* `SYSCALL_METADATA`;
* `__SYSCALL_DEFINEx`.

 Về macro đầu tiên `SYSCALL_METADATA`, nó phụ thuộc vào tham số cấu hình tên `CONFIG_FTRACE_SYSCALLS` của kernel.  Từ tên, chúng ta cũng có thể hiểu được phần nào về tham số cấu hình này, nó cho phép truy vết sự kiện ở cửa vào (entry - khi bắt đầu được gọi) và cửa ra (exit - khi kết thúc). Nếu tham số cấu hình này được bật, macro `SYSCALL_METADATA` sẽ thực hiện việc khởi tạo cấu trúc dữ liệu `syscall_metadata` được định nghĩa trong file header [include/trace/syscall.h](https://github.com/torvalds/linux/blob/master/include/trace/syscall.h), chứa các trường dành cho việc truy vết như tên của lời gọi, chỉ số của lời gọi system call trong bảng lời gọi (system call [table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)), số lượng tham số của một lời gọi (system call), danh sách các tham số, vân vân:

```C
#define SYSCALL_METADATA(sname, nb, ...)                             \
	...                                                              \
	...                                                              \
	...                                                              \
    struct syscall_metadata __used                                   \
              __syscall_meta_##sname = {                             \
                    .name           = "sys"#sname,                   \
                    .syscall_nr     = -1,                            \
                    .nb_args        = nb,                            \
                    .types          = nb ? types_##sname : NULL,     \
                    .args           = nb ? args_##sname : NULL,      \
                    .enter_event    = &event_enter_##sname,          \
                    .exit_event     = &event_exit_##sname,           \
                    .enter_fields   = LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
             };                                                                            \

    static struct syscall_metadata __used                           \
              __attribute__((section("__syscalls_metadata")))       \
             *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```

 Nếu tham số cấu hình nhân `CONFIG_FTRACE_SYSCALLS` không được bật khi cấu hình nhân, macro `SYSCALL_METADATA` trở thành một chuỗi rỗng:

```C
#define SYSCALL_METADATA(sname, nb, ...)
```

Macro thứ 2 `__SYSCALL_DEFINEx` mở rộng thành định nghĩa của 5 hàm sau:

```C
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       \
                __attribute__((alias(__stringify(SyS##name))));         \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
        {                                                               \
                long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
                __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                return ret;                                             \
        }                                                               \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

 Đầu tiên `sys##name` định nghĩa hàm handle cho lời gọi (syscall handler function) theo tên được truyền vào - `sys_system_call_name`. Macro `__SC_DECL` sẽ lấy tham số `__VA_ARGS__` rồi kết hợp kiểu tên của tham số đầu vào lời gọi, bởi vì bởi vì macro hiện tại không thể phát hiện được kiểu của tham số. Cuối cùng macro `__MAP` gọi macro `__SC_DECL` với tham số là macro `__VA_ARGS__`. Các hàm khác được sinh ra bởi macro `__SYSCALL_DEFINEx` là cần thiết để bảo vệ lỗi hổng có mã [CVE-2009-0029](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-0029) và chúng ta sẽ không đi vào chi tiết ở đây. Ok, kết quả của macro `SYSCALL_DEFINE3` sau khi mở rộng sẽ như sau:

```C
asmlinkage long sys_write(unsigned int fd, const char __user * buf, size_t count);
```

 Giờ chúng ta cũng đã biết một chút về định nghĩa lời gọi hệ thống rồi và chúng chúng ta trở lại thực thi của lời gọi `write`. Cùng nhau nhìn thực thì của lời gọi này một lần nữa:

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

 Như chúng ta đã biết, cũng như thấy ngay từ source code, lời gọi trên nhận 3 tham số:

* `fd`    - file descriptor;
* `buf`   - buffer để chứa dữ liệu cần ghi;
* `count` - độ dài của dữ liệu cần ghi.

 rồi ghi dữ liệu chứa trong bufer được đưa vào bởi user vào một thiết bị hoặc file. Chú ý rằng tham số thứ 2 `buf`, được định nghĩa với thuộc tính `__user`. Mục đích chính của thuộc tính này là dành cho việc kiểm tra code nhân Linux bằng công cụ có tên [sparse](https://en.wikipedia.org/wiki/Sparse). Nó được định nghĩa ở đây file header[include/linux/compiler.h](https://github.com/torvalds/linux/blob/master/include/linux/compiler.h) và phụ thuộc vào định nghĩa `__CHECKER__` trong nhân Linux. Đến đây, ta đã nói về tất cả những thông tin hữu ích (useful meta-information) liên quan đến lời gọi `sys_write`, cùng nhau hiểu làm thế nào làm thế nào lời gọi được thực thi (implement).  Chúng ta bắt đầu từ dòng khai báo biến `f` có kiểu cấu trúc `fd` biểu diễn một file descriptor trong nhân Linux và giá trị được gán cho biến này được thực hiện thông qua hàm `fdget_pos`. Hàm `fdget_pos` được định nghĩa trong file source [source](https://github.com/torvalds/linux/blob/master/fs/read_write.c), nó đơn giản là sử dụng tiếp hàm `__to_fd`:

```C
static inline struct fd fdget_pos(int fd)
{
        return __to_fd(__fdget_pos(fd));
}
```

 Mục đích chính của hàm `fdget_pos` chuyển đổi cái file descriptor hay chỉ là một con số sang cấu trúc `fd`. Thông qua một dãy dài các lời gọi khác nữa, hàm `fdget_pos` sẽ thực hiện lấy bảng file descriptor của tiến trình hiện tại, `current->files`, và cố gắng tìm giá trị tương ứng với con số được gán ở file descriptor. Khi đã có được giá trị cấu trúc `fd` tương ứng cho file descriptor, chúng ta sẽ kiểm tra, trả về ngay nếu không tồn tại. Nếu xử lý tiếp tục, chúng ta sẽ lấy vị trị hiện tại trong file sắp được ghi thông qua hàm `file_pos_read`, cái hàm đơn giản là trả về giá trị trường `f_pos` cho file đó:

```C
static inline loff_t file_pos_read(struct file *file)
{
        return file->f_pos;
}
```

 Sau đó, gọi hàm `vfs_write`. Hàm `vfs_write` được định nghĩa trong file source code [fs/read_write.c](https://github.com/torvalds/linux/blob/master/fs/read_write.c) và sẽ giúp chúng ta thực hiện thao tác ghi - tức là ghi một buffer vào một file theo từ vị trí nào đó . Chúng ta sẽ không đi chi tiết vào hàm `vfs_write`, bởi vì hàm này ít liên quan đến khái niệm `system call`, chúng thuộc các khái niệm liên quan đến [Virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system), cái ta sẽ xem xét ở một chương khác. Sau khi hàm `vfs_write` thực hiện xong công việc của nó, chúng ta sẽ kiểm tra kết quả xem nó có ghi thành công hay không; nếu thành công, chúng ta sẽ thay đổi vị trí con trỏ trong file thông qua hàm `file_pos_write`:

```C
if (ret >= 0)
	file_pos_write(f.file, pos);
```

 đơn giản là thực hiện việc cập nhật giá trị `f_pos` đến giá trị mới cho file chúng ta vừa ghi:

```C
static inline void file_pos_write(struct file *file, loff_t pos)
{
        file->f_pos = pos;
}
```

 Đoạn kết thúc handle của lời gọi `write`, chúng ta sẽ thấy lời gọi thực hiện như sau:

```C
fdput_pos(f);
```

 để unlock mutext `f_pos_lock` cái bảo vệ vị trí trong file trong suốt quá trình khi file được sử dụng bởi nhiều thread chia sẽ file descriptor.

 Tất cả chỉ có vậy.

 Chúng ta vừa xem xét cụ thể thực thi của một lời gọi hệ thống trong nhân Linux. Tất nhiên, chúng ta đã không xem xét kĩ một vài chỗ trong lời gọi `write` bởi vì như tôi đã nói ở trên, chúng ta chỉ để ý đến những thứ liên quan đến khái niệm lời gọi hệ thống trong chương này thôi, và không đề cấp chi tiết đến các hệ thống con khác như [Virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system).

 Kết bài
--------------------------------------------------------------------------------

 Đây là chương đầu tiên trong loạt tìm hiểu về các khái niệm lời gọi hệ thống (system call concepts) trong nhân Linux. Chúng ta sẽ tiếp tục nói về lời gọi hệ thống thêm nữa và trong chương tiếp theo chúng ta lại tiếp tục tìm hiểu vấn đề này thông qua source code của nhân Linux.

 Nếu bạn có bất cứ câu hỏi nào, hãy thoải mái hỏi tôi trên [0xAX](https://twitter.com/0xAX), qua [email](anotherworldofworld@gmail.com) hoặc đơn giản là tạo một [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [vdso](https://en.wikipedia.org/wiki/VDSO)
* [vsyscall](https://lwn.net/Articles/446528/)
* [general purpose registers](https://en.wikipedia.org/wiki/Processor_register)
* [socket](https://en.wikipedia.org/wiki/Network_socket)
* [C programming language](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [x86](https://en.wikipedia.org/wiki/X86)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [x86-64 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)
* [System V Application Binary Interface. PDF](http://www.x86-64.org/documentation/abi.pdf)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Intel manual. PDF](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [system call table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)
* [GCC macro documentation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29)
* [strace](https://en.wikipedia.org/wiki/Strace)
* [standard library](https://en.wikipedia.org/wiki/GNU_C_Library)
* [wrapper functions](https://en.wikipedia.org/wiki/Wrapper_function)
* [ltrace](https://en.wikipedia.org/wiki/Ltrace)
* [sparse](https://en.wikipedia.org/wiki/Sparse)
* [proc file system](https://en.wikipedia.org/wiki/Procfs)
* [Virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system)
* [systemd](https://en.wikipedia.org/wiki/Systemd)
* [epoll](https://en.wikipedia.org/wiki/Epoll)
* [Previous chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)
