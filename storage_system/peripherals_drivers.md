# 外设与驱动程序

[TOC]

## 1. 总线，中断控制器和 DMA

I/O 设备的实质如下图所示：

<img src="images\device\IOdevice_abstract.png" alt="IOdevice_abstract" style="zoom:67%;" />

想要和 I/O 设备交互的程序可以读取它的状态寄存器，向 I/O 设备的命令寄存器中写入命令，从 I/O 设备的数据寄存器中写入/向其中输出数据，在 I/O 设备的内部存在着用于实现 I/O 设备的自己的功能的微处理器，以及它的存储器，以及不同类型的 I/O 设备特有的一些硬件。

总线可以理解为一个特殊的 I/O 设备，有了总线之后当计算机拥有越来越多的外部设备之后，不用再让每个外设都直接连接到 CPU，而是可以让 CPU 和总线连接，所有可以使用的外部设备通过主板上的插槽连接到总线上，外部设备通过总线和 CPU 进行通信，这样的设计相当于为 CPU 设计了一层抽象，即 CPU 可以通过总线与任何的外部设备连接，而不需要 CPU 和一个个外部设备单独连接，如下所示：

<img src="images\device\CPU_BUS_Device.png" alt="CPU_BUS_Device" style="zoom:50%;" />

外设一般有两种方式报告CPU外设的工作状态——程序查询方式和中断方式。

- **程序查询方式就是利用状态寄存器报告CPU外设的工作状态，外设只需要把其工作状态的信息填到状态寄存器里**，可惜的是状态寄存器没有能力主动告诉 CPU 它里面的值是多少，而只能被动地等待着 CPU 读取它的值。
- **中断方式要求外设具有向 CPU 发送中断请求的能力，外设每次干完活后就主动向 CPU 发中断请求，**注意是主动发中断请求，可惜的是，中断请求只能告诉 CPU 外设已经干完活，至于在干活的过程中外设是否发生错误，外设的空闲缓冲区还剩多少等其他信息无法在中断请求中表达，所以中断方式也离不开状态寄存器，CPU 响应中断后，可以读一下状态寄存器，以了解外设的更多更详细的信息。由于中断方式是主动方式，所以进程让外设干活后就可以把 CPU 时间让给别的进程，外设干完活后，中断处理程序会唤醒该进程，这就是中断方式比程序查询方式高效的原因。

DRAM 内存也可以连接到总线上，这就构成了一个简单的计算机。

早期 CPU 除了连接总线，还会连接中断控制器（比如说 Intel 8259 可编程中断控制器，可以级联以扩展中断源的数量，也可以设置中断源的优先级，也可以屏蔽特定中断源），中断控制器连接着中断源。

现代计算机中一般存在着如下两种高级中断控制器（Advanced PIC）

- **local APIC**：每个 CPU 都有一个，时钟中断由它来处理，处理器与处理器之间的中断（IPI，Inter Processor Interrupt）也由它来处理（IPI 的应用场景：多核的计算机系统在 reset 的时候只有一个核会启动，这个核会发出 IPI 将其他处理器唤醒；在两个核上运行的两个线程 T1 和 T2 ，如果 T1 线程调用了 `mmap` 系统调用，那么 T1 所在的 CPU 的 TLB 就需要被刷新，但是 T2 和 T1 是共享地址空间的，此时 T2 所在核的 TLB 没有被刷新，如果此时 T2 想要访问地址空间里刚刚 `mmap` 的那一片区域，就会发生 page falut，因此为了避免这种情况，我们需要让 T1 所在核向 T2 所在核发送一个 IPI，从而让 T2 所在核刷新 TLB，这也被称为 TLB shoot down，在具有很多个 CPU 的计算机系统当中，TLB shoot down 会造成一些性能问题）
- **I/O APIC**：整个计算机系统中只有一个，连接到外部的中断线，如果总线上有中断请求的话会传给 I/O APIC，I/O 中断就由它来处理

DMA 是除了**总线和中断以外的一种非常特殊的 I/O 设备，DMA 解决了一个中断没能完全解决的问题：如何更快完成大规模数据拷贝**。

假设程序想拷贝 1GB 的数据到磁盘，相对于 CPU，总线很慢，如果我们想让程序以循环的方式一点一点通过总线把数据拷贝到磁盘，那么开销会非常巨大。

如果系统当中有另外一个比较 Tiny 的 CPU，它只负责执行 memory copy（从内存读一个字节到这个 Tiny CPU，再从这个 Tiny CPU 把刚刚读到的字节传入总线，反过来也需要做到，从内存到内存的数据拷贝也需要做到），不需要支持某个完整的 ISA，当计算机系统的某个 CPU 正在执行一个比较耗时且简单的操作的时候，可以把这个操作交给刚刚说的 Tiny CPU 来做，等到 Tiny CPU 把任务做完了再给最开始执行任务的 CPU 的发中断。

DMA  控制器是一种在**系统内部**转移数据的独特外设，可以将其视为一种能够通过一组专用总线将内部和外部存储器与每个具有 DMA 能力的外设连接起来的控制器。**因此 DMA 的本质是通过在系统里增加一个 CPU 来加速从内存到总线的数据搬运，DMA 的外接线直接被连接到I/O总线和内存上。**



## 2. 设备驱动原理

在 CPU 以及操作系统的眼中，外部的 I/O 设备其实就是一组寄存器和一组相应的协议（规定给哪个寄存器写入哪种数据，I/O 设备会做出什么样的反应），不同的设备有着不同的协议，**外部设备本身就存在着一定的复杂性**（比如说功能很多的打印机），操作系统直接把设备以寄存器的形式暴露给应用程序，让应用程序按照外部设备的协议来编写并不是一个很好的选择（这会很大程度上提升程序编写的复杂度，出错后会造成很严重的后果，比如说打印机打印错误浪费很多纸张），因此**需要对设备做抽象，让应用程序无需访问设备的寄存器，只需要通过一个尽可能通用的API来访问设备。**

**设备驱动程序存在的意义就是把所有的 I/O 设备共有的功能提取出来，使得应用程序可以用同样的接口，屏蔽掉复杂的细节，从而完成对I/O设备的抽象。**

在 Unix/Linux 操作系统的世界当中，外部设备分为两大类：**character device** 和 **block device**，character device 是一个字节流（Byte Stream），不同于内存或者磁盘这种 read-only 情况下连续两次读取同一个位置会得到相同的值的设备，它有点像一个管道，我们可以从其中拉取数据，比如说鼠标和键盘，用户的按键的序列就是一个字节流，驱动程序读取键盘的按键序列这个字节流就可以得到用户的 input。与之相反，磁盘这种 block device 更像是一个字节数组（Byte Array）。显卡是一个特别的设备，显卡的控制器是一个字节流，显存是一个字节数组。

结合如上所述，对于一个设备来讲，从操作系统的视角来看，需要实现它的如下三种抽象，或者说是如下的三个系统调用：**read, write, ioctl**（即 i/o control，用来读取或设置设备的状态，更简单的说，用来配置设备：每个设备都有一些属于它的特殊的控制选项，比如说显卡可以设置它的分辨率，打印机可以人为控制它的纸盘，键盘可以开启跑马灯、键盘宏功能），**驱动程序的代码就是用来建立这三种抽象，从这个角度来看，驱动程序和 shell 很像，shell 会将用户输入的命令翻译成一组系统调用**。

通过 `open` 系统调用（比如说 `open("/dev/urandom")`）可以激活设备的驱动程序，并且返回一个文件描述符（假设叫 `fd`），在通过 `read(fd)` 这样的系统调用从其中读数据时，操作系统就会调用这个设备相应的驱动程序。

Unix 系统的 `/dev` 目录中的设备不都是真实存在的，比如说 `dev/null`（其实 `dev/urandom` 也是，它可能是真正可以生成随机数的硬件，也可能是用软件来模拟的），如果某些程序的输出我们不想要了，那么就可以把它重定向到 `/dev/null`，`/dev/null` 这个现实中不存在的设备的驱动程序的实现很简单，对任何通过 `write` 调用传来的写请求，它立刻返回写入成功，对于 `read` 系统调用传来的读请求立刻返回 0。

**设备驱动程序是 Linux 内核中最多也是质量最低的代码（稍有不慎就有可能因为指针的问题 kernel panic，并且由于某些设备比较稀有导致对它的驱动程序进行测试的场景与机会都不多），目前Linux系统的一大发展趋势就是将驱动程序从内核空间转移到用户空间。**



## 3. 多个进程怎样共享外设

从共享的角度划分，外设分为**共享设备**和**独占设备**。

共享设备就是在某个活没干完时，别的进程可以让该设备干别的活，如进程 A 要从硬盘读 10MB 的数据，读完 8MB 数据时，进程 B 要求硬盘读 5MB 数据给它，这时磁盘调度算法可能让硬盘先把 B 需要的 5MB 数据读给 B，回头再给 A 读最后的 2MB 数据，具有硬盘这种特点的设备就叫共享设备。

独占设备就是外设在干某个活时，一定要先干完这个活才能干别的活，如打印机正在打印进程 A 的文档，那么在打印 A 的文档的过程中，打印机不能给其他进程打印东西，否则，打印出来的东西就面目全非了，具有打印机这种特点的设备就叫独占设备。

下面，我们以打印机为例来说明多个进程怎样共享“独占设备”的。操作系统可以设置一个打印队列，准备一个打印机的驱动程序 C，打印机每打印完一个作业时，给 CPU 发中断，CPU 响应中断，转入内核态，并跳到 C 执行，C 把该作业对应的进程唤醒，从打印队列里取出一项新作业，把相关参数如待打印数据的开始地址、数据量等，填到打印机的对应寄存器里，然后发一个“开始”命令，打印机开始打印新的作业，打印完后再给 CPU 发中断，如此周而复始地工作。某个进程想打印数据是，调用相应的 API 函数 D，D 把待打印的数据组织成一个打印作业，插入到打印队列的末尾，把进程状态设为挂起状态，然后调用进程调度函数切换别的进程执行，在以后的某个时刻，该进程的作业被打印完，C随即把该进程唤醒，将进程状态设为就绪状态，该进程就能往下执行了。

下面以硬盘为例讲讲多个进程是怎样共享“共享设备”的。硬盘在其控制器上设置有一个缓冲区用来暂时保存从盘片读来的数据或从内存写过来的将要写到盘片去的数据。缓冲区的大小有限，如 8MB，而读写的文件可能很大，如一个视频文件可能有几百 MB 大，所以，一个读写作业可能需要读写多次才能完成。同样地，操作系统需要设置一个类似于刚才所说的“打印队列”的数据结构用来记录各个进程待读写的数据，需要准备一个硬盘中断处理程序E。硬盘完成一次读写后给 CPU 发中断，CPU 转入内核态并跳到 E 执行，如果是写操作，E 把硬盘缓冲区里的数据搬到内存，然后根据某种磁盘调度算法，如：先来先服务、电梯算法、最短寻道优先等算法从各个读写作业中调一个它认为最好的作业出来，并命令硬盘处理该作业。如果在某次中断处理过程中发现某个进程的待读写数据的剩余数据量为 0，则表明该进程的读写作业已经完成，E把该进程唤醒，并把进程状态设为就绪状态，该进程就能往下执行了。



## 4. Linux 上的驱动原理

假设我们要实现一个核弹发射器的驱动程序（此处不得不佩服jyy的脑洞 ，程序里定义的password可能也是个彩蛋吧2333）

```c
#include <fcntl.h>
#define SECRET "\x01\x14\x05\x14"

int main() {
    int fd = open("/dev/nuke", O_WRONLY);
    if (fd > 0) {
        write(fd, SECRET, sizeof(SECRET)-1);
        close(fd);
    } else {
        perror("launcher");
    }
}
```

jyy 老师提供的代码：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define MAX_DEV 2

static int dev_major = 0;
static struct class *lx_class = NULL;
static struct cdev cdev;

static ssize_t lx_read(struct file *, char __user *, size_t, loff_t *);
static ssize_t lx_write(struct file *, const char __user *, size_t, loff_t *);

static struct file_operations fops = {
  .owner = THIS_MODULE,
  .read = lx_read,
  .write = lx_write,
};

static struct nuke {
  struct cdev cdev;
} devs[MAX_DEV];

static int __init lx_init(void) {
  dev_t dev;
  int i;

  // allocate device range
  alloc_chrdev_region(&dev, 0, 1, "nuke");

  // create device major number
  dev_major = MAJOR(dev);

  // create class
  lx_class = class_create(THIS_MODULE, "nuke");

  for (i = 0; i < MAX_DEV; i++) {
    // register device
    cdev_init(&devs[i].cdev, &fops);
    cdev.owner = THIS_MODULE;
    cdev_add(&devs[i].cdev, MKDEV(dev_major, i), 1);
    device_create(lx_class, NULL, MKDEV(dev_major, i), NULL, "nuke%d", i);
  }
  return 0;    
}

static void __exit lx_exit(void) {
  device_destroy(lx_class, MKDEV(dev_major, 0));
  class_unregister(lx_class);
  class_destroy(lx_class);
  unregister_chrdev_region(MKDEV(dev_major, 0), MINORMASK);
}

static ssize_t lx_read(struct file *file, char __user *buf, size_t count, loff_t *offset) {
  if (*offset != 0) {
    return 0;
  } else {
    uint8_t *data = "This is dangerous!\n";
    size_t datalen = strlen(data);
    if (count > datalen) {
      count = datalen;
    }
    if (copy_to_user(buf, data, count)) {
      return -EFAULT;
    }
    *offset += count;
    return count;
  }
}

static ssize_t lx_write(struct file *file, const char __user *buf, size_t count, loff_t *offset) {
  char databuf[4] = "\0\0\0\0";
  if (count > 4) {
    count = 4;
  }

  copy_from_user(databuf, buf, count);
  if (strncmp(buf, "\x01\x14\x05\x14", 4) == 0) {
    const char *EXPLODE[] = {
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣀⣀⠀⠀⣀⣤⣤⣄⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⣀⣠⣤⣤⣾⣿⣿⣿⣿⣷⣾⣿⣿⣿⣿⣿⣶⣿⣿⣿⣶⣤⡀⠀⠀⠀⠀",
      "    ⠀⢠⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣷⠀⠀⠀⠀",
      "    ⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣶⡀⠀",
      "    ⠀⢀⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡇⠀",
      "    ⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡿⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⠟⠁⠀",
      "    ⠀⠀⠻⢿⡿⢿⣿⣿⣿⣿⠟⠛⠛⠋⣀⣀⠙⠻⠿⠿⠋⠻⢿⣿⣿⠟⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠈⠉⣉⣠⣴⣷⣶⣿⣿⣿⣿⣶⣶⣶⣾⣶⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠉⠛⠋⠈⠛⠿⠟⠉⠻⠿⠋⠉⠛⠁⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣶⣷⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⢀⣀⣠⣤⣤⣤⣤⣶⣿⣿⣷⣦⣤⣤⣤⣤⣀⣀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⢰⣿⠛⠉⠉⠁⠀⠀⠀⢸⣿⣿⣧⠀⠀⠀⠀⠉⠉⠙⢻⣷⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠙⠻⠷⠶⣶⣤⣤⣤⣿⣿⣿⣿⣦⣤⣤⣴⡶⠶⠟⠛⠁⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣴⣿⣿⣿⣿⣿⣿⣷⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠒⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠓⠀⠀⠀⠀⠀⠀⠀⠀⠀",
    };
    int i;

    for (i = 0; i < sizeof(EXPLODE) / sizeof(EXPLODE[0]); i++) {
      printk("\033[01;31m%s\033[0m\n", EXPLODE[i]);
    }
  } else {
    printk("nuke: incorrect secret, cannot lanuch.\n");
  }
  return count;
}

module_init(lx_init);
module_exit(lx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("jyy");

```

代码最后的

```c
module_init(lx_init);
module_exit(lx_exit);
```

这一段用来标记内核中这个驱动对应的模块的起点和终点，这和最开始 include 的 Linux 内核中的 `module` 库有关，这段设备驱动在被编译完了之后会生成类似于 .so 这样的 shared object 的可以动态加载的动态链接库（实际上是 .ko : kernel object，Linux 内核启动的时候内存里可能没有这个模块，可以在 Linux 启动之后通过一个系统调用把这个模块加载到内核里），`module_init` 和 `module_exit` 这两个宏存在的意义是可以在模块 init 和 unload 的时候分别调用 `lx_init` 函数与 `lx_exit` 函数。

`lx_init` 函数会利用 Linux 内核提供的 API 把设备注册上去

```c
static int __init lx_init(void) {
  dev_t dev;
  int i;

  // allocate device range
  alloc_chrdev_region(&dev, 0, 1, "nuke");

  // create device major number
  // 给这个设备分配一个主设备的设备号
  dev_major = MAJOR(dev);

  // create class
  // 创建一个设备的类别
  lx_class = class_create(THIS_MODULE, "nuke");

  // 创建MAX_DEV个“核弹发射井”
  for (i = 0; i < MAX_DEV; i++) {
    // register device
    // 给每个设备的数据做初始化
    cdev_init(&devs[i].cdev, &fops);
    cdev.owner = THIS_MODULE;
    cdev_add(&devs[i].cdev, MKDEV(dev_major, i), 1);
    // 创建一个这个类型的设备
    // MKDEV中的第一个参数是主设备号，第二个参数是小的设备号
    // "nuke%d"是使用printf语法来指定设备名字
    device_create(lx_class, NULL, MKDEV(dev_major, i), NULL, "nuke%d", i);
  }
  return 0;    
}
```

`lx_exit` 中要执行上述操作的反操作

Linux 当中一切皆文件，因此只需要如下的 `file_operations` 结构体就可以注册设备驱动程序

```c
static struct file_operations fops = {
  .owner = THIS_MODULE,
  .read = lx_read,
  .write = lx_write,
};
```

在前面的 `lx_init` 的 `code_init` 函数中会把这种结构体作为参数传入，这样的话通过系统调用读写这个设备的时候控制流就会走到我们所注册的函数那里（可以用 `strace` 命令验证），`lx_read` 中有一些 error checking 以确保驱动程序的安全性

工业界真实的驱动程序的 `file_operations` 结构体中会注册更多的函数：

```c
struct file_operations {
	struct module *owner;
  	loff_t (*llseek) (struct file *, loff_t, int);
  	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
  	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
  	int (*mmap) (struct file *, struct vm_area_struct *);
  	unsigned long mmap_supported_flags;
  	int (*open) (struct inode *, struct file *);
  	int (*release) (struct inode *, struct file *);
  	int (*flush) (struct file *, fl_owner_t id);
  	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
  	int (*lock) (struct file *, int, struct file_lock *);
  	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
  	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
  	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
  	int (*flock) (struct file *, int, struct file_lock *);
  	...
```

其中和 ioctl 有关的函数有两个：

- **unlocked_ioctl**：BKL (Big Kernel Lock) 时代的遗产
  - 单处理器时代只有 ioctl。
  - 之后引入了 BKL，ioctl 执行时默认持有 BKL。
  - (2.6.11) 高性能的驱动可以通过 unlocked_ioctl 避免锁。
  - (2.6.36) ioctl 从struct file_operations中移除。
- **compact_ioctl**：机器字长的兼容性
  - 32-bit 程序在 64-bit 系统上可以 ioctl。
  - 此时应用程序和操作系统对 ioctl 数据结构的解读可能不同 (tty)。
  - （调用此兼容模式）。

早期使用 BKL 的架构下，在持有锁的情况下访问低速的设备会很影响性能，因此后来有了 `unlocked_ioctl` 在无锁的情况下进行 ioctl，`compact_ioctl` 存在的意义是为了在 64 位机器上运行 32 位的程序，因为 32 位的程序中的 ioctl 是 32 位的，涉及指针相关的时候就会有很多麻烦。

