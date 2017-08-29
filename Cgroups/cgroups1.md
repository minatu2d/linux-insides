Nhóm điều khiển (Control Groups)
================================================================================

Giới thiệu
--------------------------------------------------------------------------------

Đây là phần đầu tiên trong chương 2 của cuốn [linux insides](http://0xax.gitbooks.io/linux-insides/content/), và như bạn cũng thấy từ cái tên của nó, phần này sẽ nói về [control groups](https://en.wikipedia.org/wiki/Cgroups) hay cơ chế `cgroups` bên trong nhân Linux.

`Cgroups` là cơ chế đặc biệt được cung cấp bởi nhân Linux cho phép chúng ta cấp phát những thứ gọi là `tài nguyên` (`resources`) như là thời gian sử dụng bộ xử lý (processor time), số lượng tiếng trình trong nhóm (number of processes per group), ượng bộ nhớ RAM trên mỗi nhóm (amount of memory per control group) hoặc kết hợp các tài nguyên như trên cho một tiến trình, nhóm tiến trình. `Cgroups` được tổ chức dạng phân lớp và ở đây, nó tương tự cơ chế với tiến trình thông thường (usual processes) và `cgroups` con kế thừa tập tham số từ cha của chúng. Nhưng thực sự có một chút khác biệt ở đây. Sự khác biệt chủ yếu giữa `cgroups` and tiến trình thông thường là rất nhiều phân lớp khác nhau của `cgroup` có thể tồn tại đồng thời. Trong khi cây tiến trình thì chỉ có 1 mà thôi. Đây không phải là điều tự nó thế, mà bởi vì mỗi nhóm điều khiển (cgroup) được gắn vào một tập của các cgroup (set of control group), hay `hệ thống con` dành cho nhóm điều khiển (control group `subsystems`).

Một `control group subsystem` biểu diễn một loại tài nguyên như thời gian sử dụng bộ xử lý (processor time) hoặc một số lượng các [pids](https://en.wikipedia.org/wiki/Process_identifier) hay nói cách khác là số lượng tiến trình cho một `control group`. Nhân Linux kernel cung cấp đến 12 loại hệ thống con nhóm điều khiển dưới đây (`control group subsystems`)

* `cpuset` - gán processor(một hoặc nhiều) và node bộ nhớ (memory nodes) cho task (1 hoặc nhiều) trong 1 nhóm;
* `cpu` - sử dụng bộ lập lịch đẻ cung cấp truy cập task của nhóm đến tài nguyên xử lý (processor resource);
* `cpuacct` - sinh một báo cáo về việc sử dụng processor cho 1 nhóm;
* `io` - thiết lập giới hạn đọc ghi từ/đến [block devices](https://en.wikipedia.org/wiki/Device_file);
* `memory` - thiết lập giới hạn sử dụng bộ nhớ bởi task(1 hoặc nhiều) trong 1 nhóm;
* `devices` - cho phép việc truy cập thiết bị từ task(một hoặc nhiều) trong 1 nhóm;
* `freezer` - cho phép tạm dừng/chạy tiếp cho task (một hoặc nhiều) trong 1 nhóm;
* `net_cls` - cho phép đánh dấu gói dữ liệu mạng từ task (một hoặc nhiều) trong 1 nhóm;
* `net_prio` - cung cấp một cách cho phép thiết lập động độ ưu tiên trên mỗi giao thức mạng cho một nhóm;
* `perf_event` - cung cấp truy cập đến [perf events](https://en.wikipedia.org/wiki/Perf_(Linux)) cho 1 nhóm;
* `hugetlb` - kích hoạt khả năng hỗ trợ [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt) cho nhóm;
* `pid` - thiết lập giới hạn số tiến trình có trong 1 nhóm.

Hoạt động của mỗi hệ thống con ở trên phụ thuộc vào tham số cấu hình của nó. Ví dụ hệ thống con `cpuset` được bật thông qua tham số cấu hình nhân `CONFIG_CPUSETS`, hệ thống con `io` cũng được bật thông qua tham số cấu hình nhân `CONFIG_BLK_CGROUP` hay những hệ thống con tương tự. Tất cả các cấu hình nhân này có thể thấy ở phần menu `General setup → Control Group support` khi build nhân:

![menuconfig](http://oi66.tinypic.com/2rc2a9e.jpg)

Bạn có thể thấy các nhóm điều khiển thông qua hệ thống file [proc](https://en.wikipedia.org/wiki/Procfs):

```
$ cat /proc/cgroups 
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	8	1	1
cpu	7	66	1
cpuacct	7	66	1
blkio	11	66	1
memory	9	94	1
devices	6	66	1
freezer	2	1	1
net_cls	4	1	1
perf_event	3	1	1
net_prio	4	1	1
hugetlb	10	1	1
pids	5	69	1
```

hoặc thông qua hệ thống file sysfs [sysfs](https://en.wikipedia.org/wiki/Sysfs):

```
$ ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 5 root root  0 Dec  2 22:37 blkio
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpuacct -> cpu,cpuacct
dr-xr-xr-x 5 root root  0 Dec  2 22:37 cpu,cpuacct
dr-xr-xr-x 2 root root  0 Dec  2 22:37 cpuset
dr-xr-xr-x 5 root root  0 Dec  2 22:37 devices
dr-xr-xr-x 2 root root  0 Dec  2 22:37 freezer
dr-xr-xr-x 2 root root  0 Dec  2 22:37 hugetlb
dr-xr-xr-x 5 root root  0 Dec  2 22:37 memory
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_cls -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_prio -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 perf_event
dr-xr-xr-x 5 root root  0 Dec  2 22:37 pids
dr-xr-xr-x 5 root root  0 Dec  2 22:37 systemd
```

Bạn có thể đã đoán ra rằng cơ chế `control groups` không phải là cơ chế liên quan trực tiếp đến yêu cầu từ phía nhân Linux, mà nó được dùng hầu hết cho phía user-space. Để sử dụng `control group`, chúng ta phải tạo nó. Việc tạo `cgroup` được thực hiện thông qua 2 cách sau.

Các đầu tiên, là tạo một thư mục con trong bất cứ thư mục hệ thống con nào nằm ở `sys/fs/cgroup` và thêm pid (process id) vào file có tên là `tasks`, file này được tạo tự động khi chúng ta tạo thư mục con.

Các thứ hai, là nằm trong nhóm chức năng create/destroy/manage `cgroups` được cung cấp thông qua các công cụ từ thư viện `libcgroup` (trong Fedora thư viện này có tên là `libcgroup-tools`).

Nào cùng nhau xem xét một ví dụ. Đoạn script [bash](https://www.gnu.org/software/bash/) dưới đây sẽ ghi một dòng vào thiết bị có đường dẫn `/dev/tty`, chính là thiết bị điều khiển termial (màn hình chạy lệnh) cho tiến trình hiện tại luôn:

```shell
#!/bin/bash

while :
do
    echo "print line" > /dev/tty
    sleep 5
done
```

Vì thế, khi bạn chạy script này, bạn sẽ thấy kết quả như sau:

```
$ sudo chmod +x cgroup_test_script.sh
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
...
```

Nào, giờ cùng nhau đến xem chỗ mà `cgroupfs` được gắn vào máy tính của bạn. Như bạn đã thấy ở trên, đó là thư mục `/sys/fs/cgroup`, tất nhiên bạn có thể gắn chúng ở bất cứ đâu.

```
$ cd /sys/fs/cgroup
```

Và bây giờ, chúng ta vào xem thư mục con có tên `devices` để xem loại resource nào được phép/không được phép truy cập vào thiết bị bởi các task có trong `cgroup`:

```
# cd /devices
```

và tạo tiếp thư mục `cgroup_test_group` ở đó:

```
# mkdir cgroup_test_group
```

Sau khi tạo thư mục `cgroup_test_group`, các file bên dưới đây sễ thực động được sinh ra:

```
/sys/fs/cgroup/devices/cgroup_test_group$ ls -l
total 0
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.clone_children
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.procs
--w------- 1 root root 0 Dec  3 22:55 devices.allow
--w------- 1 root root 0 Dec  3 22:55 devices.deny
-r--r--r-- 1 root root 0 Dec  3 22:55 devices.list
-rw-r--r-- 1 root root 0 Dec  3 22:55 notify_on_release
-rw-r--r-- 1 root root 0 Dec  3 22:55 tasks
```

Ở thời điểm này, chúng ta chỉ quan tâm đến 2 file `tasks` và `devices.deny` mà thôi. Đầu tiên, file `tasks` chứa pid của các tiến trình sẽ được gắn vào group ta mới tạo, tên `cgroup_test_group`. Thứ 2, file `devices.deny` chứa danh sách các thiết bị không được phép truy cập. Mặc định, group mới tạo không có bất cứ giới hạn nào cho việc truy cập thiết bị. Để cấm truy cập một thiết bị (trong trường hợp này giả sử là `/dev/tty`) chúng ta sẽ viết vào file `devices.deny` dòng sau đây:

```
# echo "c 5:0 w" > devices.deny
```

Trong có vẻ ma thuật, chúng ta sẽ phân tích dòng lệnh này. Chứ `c` đầu tiên sau `echo` biểu diễn loại thiết bị. Vì thiết bị chúng ta đã nói đến `/dev/tty` là `char device` mà nên giá trị ở đó là `c`. Chúng ta có thể kiểm tra bất cứ thiết bị nào bằng lênh phổ biến `ls`:

```
~$ ls -l /dev/tty
crw-rw-rw- 1 root tty 5, 0 Dec  3 22:48 /dev/tty
```

Bạn thấy chữ `c` đầu tiên trong danh sách quyền đọc/ghi/chạy chứ. Phần thứ 2 ta cũng thấy được là `5:0`, đây chính là số minor và major numbers của thiết bị. Như bạn cũng, nó cũng xuất hiện trong kết quả khi chạy lệnh `ls`. Kí tự cuối cùng `w`, tức là cấm các task thực hiện quyền ghi đến thiết bị. Nào ta thử thực hiện script `cgroup_test_script.sh` sau:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
```

giờ thêm pid của tiến trình hiện tại (tức là terminal) vào file `devices/tasks` của group:

```
# echo $(pidof -x cgroup_test_script.sh) > /sys/fs/cgroup/devices/cgroup_test_group/tasks
```

Kết quả của hành động này sẽ như sau:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
print line
print line
print line
./cgroup_test_script.sh: line 5: /dev/tty: Operation not permitted
```

Điều này được sử dụng trong [docker](https://en.wikipedia.org/wiki/Docker_(software)) containers, ví dụ:

```
~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
fa2d2085cd1c        mariadb:10          "docker-entrypoint..."   12 days ago         Up 4 minutes        0.0.0.0:3306->3306/tcp   mysql-work

~$ cat /sys/fs/cgroup/devices/docker/fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61/tasks | head -3
5501
5584
5585
...
...
...
```

Vì thế, trong suốt quá trình khởi động của một `docker`, `docker` sẽ tạo một `cgroup` cho các tiến trình trong container đó:

```
$ docker exec -it mysql-work /bin/bash
$ top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                   1 mysql     20   0  963996 101268  15744 S   0.0  0.6   0:00.46 mysqld                                                                                  71 root      20   0   20248   3028   2732 S   0.0  0.0   0:00.01 bash                                                                                    77 root      20   0   21948   2424   2056 R   0.0  0.0   0:00.00 top                                                                                  
```

Bạn có thể thấy các `cgroup` này trên máy host:

```C
$ systemd-cgls

Control group /:
-.slice
├─docker
│ └─fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61
│   ├─5501 mysqld
│   └─6404 /bin/bash
```

Giờ, chúng ta đã hiểu một chút về cơ chế `control groups` rồi, vậy nó được sử dụng như thế nào và mục đích của cơ chế này là gì. Giờ là lúc để tiếp túc xem xét mã nguồn nhân Linux và bắt đầu phân tích xử lý của cơ chế này.

Early initialization of control groups
--------------------------------------------------------------------------------

Now after we just saw little theory about `control groups` Linux kernel mechanism, we may start to dive into the source code of Linux kernel to acquainted with this mechanism closer. As always we will start from the initialization of `control groups`. Initialization of `cgroups` divided into two parts in the Linux kernel: early and late. In this part we will consider only `early` part and `late` part will be considered in next parts.

Early initialization of `cgroups` starts from the call of the:

```C
cgroup_init_early();
```

function in the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) during early initialization of the Linux kernel. This function is defined in the [kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c) source code file and starts from the definition of two following local variables:

```C
int __init cgroup_init_early(void)
{
	static struct cgroup_sb_opts __initdata opts;
	struct cgroup_subsys *ss;
    ...
    ...
    ...
}
```

The `cgroup_sb_opts` structure defined in the same source code file and looks:

```C
struct cgroup_sb_opts {
	u16 subsys_mask;
	unsigned int flags;
	char *release_agent;
	bool cpuset_clone_children;
	char *name;
	bool none;
};
```

which represents mount options of `cgroupfs`. For example we may create named cgroup hierarchy (with name `my_cgrp`) with the `name=` option and without any subsystems:

```
$ mount -t cgroup -oname=my_cgrp,none /mnt/cgroups
```

The second variable - `ss` has type - `cgroup_subsys` structure which is defined in the [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup-defs.h) header file and as you may guess from the name of the type, it represents a `cgroup` subsystem. This structure contains various fields and callback functions like:

```C
struct cgroup_subsys {
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    ...
    ...
    ...
    bool early_init:1;
    int id;
    const char *name;
    struct cgroup_root *root;
    ...
    ...
    ...
}
```

Where for example `ccs_online` and `ccs_offline` callbacks are called after a cgroup successfully will complet all allocations and a cgroup will be before releasing respectively. The `early_init` flags marks subsystems which may/should be initialized early. The `id` and `name` fields represents unique identifier in the array of registered subsystems for a cgroup and `name` of a subsystem respectively. The last - `root` fields represents pointer to the root of of a cgroup hierarchy.

Of course the `cgroup_subsys` structure bigger and has other fields, but it is enough for now. Now as we got to know important structures related to `cgroups` mechanism, let's return to the `cgroup_init_early` function. Main purpose of this function is to do early initialization of some subsystems. As you already may guess, these `early` subsystems should have `cgroup_subsys->early_init = 1`. Let's look what subsystems may be initialized early.

After the definition of the two local variables we may see following lines of code:

```C
init_cgroup_root(&cgrp_dfl_root, &opts);
cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;
```

Here we may see call of the `init_cgroup_root` function which will execute initialization of the default unified hierarchy and after this we set `CSS_NO_REF` flag in state of this default `cgroup` to disable reference counting for this css. The `cgrp_dfl_root` is defined in the same source code file:

```C
struct cgroup_root cgrp_dfl_root;
```

Its `cgrp` field represented by the `cgroup` structure which represents a `cgroup` as you already may guess and defined in the [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup-defs.h) header file. We already know that a process which is represented by the `task_struct` in the Linux kernel. The `task_struct` does not contain direct link to a `cgroup` where this task is attached. But it may be reached via `ccs_set` field of the `task_struct`. This `ccs_set` structure holds pointer to the array of subsystem states:

```C
struct css_set {
    ...
    ...
    ....
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    ...
    ...
    ...
}
```

And via the `cgroup_subsys_state`, a process may get a `cgroup` that this process is attached to:

```C
struct cgroup_subsys_state {
    ...
    ...
    ...
    struct cgroup *cgroup;
    ...
    ...
    ...
}
```

So, the overall picture of `cgroups` related data structure is following:

```                                                 
+-------------+         +---------------------+    +------------->+---------------------+          +----------------+
| task_struct |         |       css_set       |    |              | cgroup_subsys_state |          |     cgroup     |
+-------------+         |                     |    |              +---------------------+          +----------------+
|             |         |                     |    |              |                     |          |     flags      |
|             |         |                     |    |              +---------------------+          |  cgroup.procs  |
|             |         |                     |    |              |        cgroup       |--------->|       id       |
|             |         |                     |    |              +---------------------+          |      ....      | 
|-------------+         |---------------------+----+                                               +----------------+
|   cgroups   | ------> | cgroup_subsys_state | array of cgroup_subsys_state
|-------------+         +---------------------+------------------>+---------------------+          +----------------+
|             |         |                     |                   | cgroup_subsys_state |          |      cgroup    |
+-------------+         +---------------------+                   +---------------------+          +----------------+
                                                                  |                     |          |      flags     |
                                                                  +---------------------+          |   cgroup.procs |
                                                                  |        cgroup       |--------->|        id      |
                                                                  +---------------------+          |       ....     |
                                                                  |    cgroup_subsys    |          +----------------+
                                                                  +---------------------+
                                                                             |
                                                                             |
                                                                             ↓
                                                                  +---------------------+
                                                                  |    cgroup_subsys    |
                                                                  +---------------------+
                                                                  |         id          |
                                                                  |        name         |
                                                                  |      css_online     |
                                                                  |      css_ofline     |
                                                                  |        attach       |
                                                                  |         ....        |
                                                                  +---------------------+
```



So, the `init_cgroup_root` fills the `cgrp_dfl_root` with the default values. The next thing is assigning initial `ccs_set` to the `init_task` which represents first process in the system:

```C
RCU_INIT_POINTER(init_task.cgroups, &init_css_set);
```

And the last big thing in the `cgroup_init_early` function is initialization of `early cgroups`. Here we go over all registered subsystems and assign unique identity number, name of a subsystem and call the `cgroup_init_subsys` function for subsystems which are marked as early:

```C
for_each_subsys(ss, i) {
		ss->id = i;
		ss->name = cgroup_subsys_name[i];

        if (ss->early_init)
			cgroup_init_subsys(ss, true);
}
```

The `for_each_subsys` here is a macro which is defined in the [kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c) source code file and just expands to the `for` loop over `cgroup_subsys` array. Definition of this array may be found in the same source code file and it looks in a little unusual way:

```C
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
    static struct cgroup_subsys *cgroup_subsys[] = {
        #include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

It is defined as `SUBSYS` macro which takes one argument (name of a subsystem) and defines `cgroup_subsys` array of cgroup subsystems. Additionally we may see that the array is initialized with content of the [linux/cgroup_subsys.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup_subsys.h) header file. If we will look inside of this header file we will see again set of the `SUBSYS` macros with the given subsystems names:

```C
#if IS_ENABLED(CONFIG_CPUSETS)
SUBSYS(cpuset)
#endif

#if IS_ENABLED(CONFIG_CGROUP_SCHED)
SUBSYS(cpu)
#endif
...
...
...
```

This works because of `#undef` statement after first definition of the `SUBSYS` macro. Look at the `&_x ## _cgrp_subsys` expression. The `##` operator concatenates right and left expression in a `C` macro. So as we passed `cpuset`, `cpu` and etc., to the `SUBSYS` macro, somewhere `cpuset_cgrp_subsys`, `cp_cgrp_subsys` should be defined. And that's true. If you will look in the [kernel/cpuset.c](https://github.com/torvalds/linux/blob/master/kernel/cpuset.c) source code file, you will see this definition:

```C
struct cgroup_subsys cpuset_cgrp_subsys = {
    ...
    ...
    ...
	.early_init	= true,
};
```

So the last step in the `cgroup_init_early` function is initialization of early subsystems with the call of the `cgroup_init_subsys` function. Following early subsystems will be initialized:

* `cpuset`;
* `cpu`;
* `cpuacct`.

The `cgroup_init_subsys` function does initialization of the given subsystem with the default values. For example sets root of hierarchy, allocates space for the given subsystem with the call of the `css_alloc` callback function, link a subsystem with a parent if it exists, add allocated subsystem to the initial process and etc.

That's all. From this moment early subsystems are initialized.

Conclusion
--------------------------------------------------------------------------------

It is the end of the first part which describes introduction into `Control groups` mechanism in the Linux kernel. We covered some theory and the first steps of initialization of stuffs related to `control groups` mechanism. In the next part we will continue to dive into the more practical aspects of `control groups`.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [control groups](https://en.wikipedia.org/wiki/Cgroups)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [cpuset](http://man7.org/linux/man-pages/man7/cpuset.7.html)
* [block devices](https://en.wikipedia.org/wiki/Device_file)
* [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [proc](https://en.wikipedia.org/wiki/Procfs)
* [cgroups kernel documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [cgroups v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
* [bash](https://www.gnu.org/software/bash/)
* [docker](https://en.wikipedia.org/wiki/Docker_(software))
* [perf events](https://en.wikipedia.org/wiki/Perf_(Linux))
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html)
