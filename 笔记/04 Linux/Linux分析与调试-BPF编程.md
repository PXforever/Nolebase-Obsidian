---
share: "true"
---

# 前言

> 我们在`BPF原理`一文中理解了BPF的使用，现在开始实践。

# 配置

编写`bpf`程序前，我们需要开启内核以下支持：

```shell
```

接着在机器上安装：

```shell
sudo apt install llvm bpf-dev
```

# 编程

> `bpf`程序分两部分：
>
> + 载入内核的代码
> + 用户层代码

我们可以参靠内核源码下的`kernel/sample/bpf/*`

![image-20240726114140248](笔记/01 附件/Linux分析与调试-BPF编程/image-20240726114140248.png)

可以看到，里面的代码大部分都是成对出现的，比如：

```shell
test_current_task_under_cgroup_kern.c
test_current_task_under_cgroup_user.c
```

我们也来编写一个这样的程序来测试。

接下来我们会在：

+ `RK3588`
+ `kernel-5.10.160`

# 编程示例

## `i2c_write`

我们先看看内核支持的事件，有这几种方式：

1. 通过`bpftool`

```shell
sudo bpftrace -l
```

2. 通过`debugfs`

```shell
# cd /sys/kernel/debug/tarcing
这里面相应的目录就是对应的事件
```

接着编写如下代码：

```c
//trace_i2c_write.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/i2c/i2c_write") //通过上面的查看获得的
int trace_i2c_write(struct pt_regs *ctx) {
    bpf_printk("i2c_write called");
    return 0;
}

char _license[] SEC("license") = "GPL";
```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_i2c_write.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_i2c_write"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_i2c_write"), "i2c", "i2c_write");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    while (1) {
        sleep(1);
    }
    
    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

`Makefile如下：`

```makefile
KER_BPF = trace_i2c_write.c
APP_BPF = loader.c

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ = ${patsubst %.c, %, ${APP_BPF}}

all: ${KER_OBJ} ${APP_OBJ}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_OBJ}: ${APP_BPF}
	gcc -o $@ $< -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_OBJ}"
	rm -rf ${KER_OBJ} ${APP_OBJ}
```

编译后，执行：

```shell
sudo ./loader
# 其中trace_i2c_write.o不需要操作，因为它是被loader加载
```

我们开启另外一个`ssh`页面，使用命令来写`i2c`：

```shell
i2cset 1  0x11 0x99 44 b
```

发现没有任何打印信息，猜想可能输出可能不会打印到标准输出，我们找到辅助函数：`trace_helpers.h`，`trace_helpers.c`。位置在：`SDK_SOURCE/kernel/tools/testing/selftests/bpf`下：

![image-20240726134805165](笔记/01 附件/Linux分析与调试-BPF编程/image-20240726134805165.png)

我们复制它俩到工程中，然后修改`Makefile`：

```makefile
APP_TARGET = loader

KER_BPF = trace_i2c_write.c
APP_BPF = loader.c trace_helpers.c 

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ =  ${patsubst %.c, %.o, ${subst ${APP_TARGET}.c, , ${APP_BPF}}}

all: ${KER_OBJ} ${APP_TARGET}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_TARGET}: ${APP_TARGET}.c ${APP_OBJ}
	gcc -o $@ $? -lbpf

${APP_OBJ}: %.o: %.c
	gcc -c $? -o $@  -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_TARGET}"
	rm -rf ${KER_OBJ} ${APP_TARGET} *.o
```

代码修改如下：

```c
//trace_i2c_write.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/i2c/i2c_write")
int trace_i2c_write(struct pt_regs *ctx) {
    bpf_printk("[bpf_demo]: i2c_write trigger\n");
    return 0;
}

char _license[] SEC("license") = "GPL";

```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_i2c_write.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_i2c_write"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_i2c_write"), "i2c", "i2c_write");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    
    read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

执行后发现打印了很多东西，也很快，我们不需要其他的数据，可以在`read_trace_pipe`中加入字符串比较进行过滤。

我们可以通过：

```shell
sudo bpftool prog list 
# 或者
sudo bpftool prog show
```
来查看事件是否插入到系统内核。
## v4l2
```makefile
APP_TARGET = loader

KER_BPF = trace_v4l2_qbuf.c
APP_BPF = loader.c trace_helpers.c 

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ =  ${patsubst %.c, %.o, ${subst ${APP_TARGET}.c, , ${APP_BPF}}}

all: ${KER_OBJ} ${APP_TARGET}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_TARGET}: ${APP_TARGET}.c ${APP_OBJ}
	gcc -o $@ $? -lbpf

${APP_OBJ}: %.o: %.c
	gcc -c $? -o $@  -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_TARGET}"
	rm -rf ${KER_OBJ} ${APP_TARGET} *.o
```

```c
//trace_v4l2_qbuf.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

#define USE_TRACE_PRINT 1

#ifdef USE_TRACE_PRINT
#include <bpf/bpf_tracing.h>
#endif

SEC("tracepoint/v4l2/v4l2_qbuf")
int trace_v4l2_qbuf(struct pt_regs *ctx) {
    unsigned long long ts = bpf_ktime_get_ns();

    unsigned long long seconds = ts / 1000000000;
    unsigned long long nanoseconds = ts % 1000000000;
#ifdef USE_TRACE_PRINT
    char fmt[] = "[bpf_demo_t]: v4l2_qbuf trigger-%llus.%llu\n";
    bpf_trace_printk( fmt, sizeof(fmt), seconds, nanoseconds);
#else
    bpf_printk("[bpf_demo]: v4l2_qbuf trigger - %llu.%09llu\n", seconds, nanoseconds);
#endif
    return 0;
}

char _license[] SEC("license") = "GPL";
```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_v4l2_qbuf.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_v4l2_qbuf"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_v4l2_qbuf"), "v4l2", "v4l2_qbuf");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
 
    while(1)
        sleep(1);
    //read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

因为在`i2c`实验中会打印大量无关的内容，我们通过查看节点来过滤：

```shell
sudo cat /sys/kernel/debug/tracing/trace_pipe |grep "bpf_demo"
```

## 输入设备事件捕获(鼠标、键盘)

代码如下：

```c
//trace_usb_hid_input.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

#define USE_TRACE_PRINT 1

#ifdef USE_TRACE_PRINT
#include <bpf/bpf_tracing.h>
#endif

SEC("kprobe/hid_input_report")
int trace_usb_hid_input(struct pt_regs *ctx) {
    static unsigned long long last_ts = 0;
    unsigned long long ts = bpf_ktime_get_ns();

#ifdef USE_TRACE_PRINT
    unsigned long long delta_ms = 0;
    unsigned long long delta_us = 0;
    char fmt[] = "[bpf_demo_t]: hid_input_report trigger - del=%llums.%lluus.%lluns\n";
    unsigned long long delta_ns = ts - last_ts;
    if( delta_ns > 1000000)
    {
        delta_ms = delta_ns/1000000;
        delta_ns %= 1000000;
    }
    if( delta_ns > 1000)
    {
        delta_us = delta_ns/1000;
        delta_ns %= 1000;
    }
    bpf_trace_printk( fmt, sizeof(fmt), delta_ms, delta_us, delta_ns);
#else
    unsigned long long seconds = ts / 1000000000;
    unsigned long long nanoseconds = ts % 1000000000; //ns
    bpf_printk("[bpf_demo]: hid_input_report trigger - %llu.%09llu\n", seconds, nanoseconds);
#endif
    last_ts = ts;
    return 0;
}

char _license[] SEC("license") = "GPL";

```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_usb_hid_input.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_usb_hid_input"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_kprobe(bpf_object__find_program_by_name(obj, "trace_usb_hid_input"), false, "hid_input_report");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    
    
    while(1)
        sleep(1);
    
    //read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

接着编译，启动，然后读取`trace`:

```shell
 cat /sys/kernel/debug/tracing/trace_pipe |grep "hid_input_report"
```

![image-20240726172411805](笔记/01 附件/Linux分析与调试-BPF编程/image-20240726172411805.png)
# 交叉编译
## 问题1
https://lore.kernel.org/bpf/20211021123913.48833-1-pulehui@huawei.com/t/
> bpf程序的交叉编译主要依赖于`rootfs`，所以如果是`buildroot`，请先看看其是否支持`libbpf`。