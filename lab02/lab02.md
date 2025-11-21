# 实验02：系统调用

## 实验目的

- 理解操作系统中系统调用的基本原理与工作机制。
- 掌握在 Linux 0.11 内核中添加自定义系统调用的方法。
- 熟悉用户态程序与内核态之间的数据传递方式（如 `get_fs_byte` 和 `put_fs_byte`）。

## 实验内容

- 在 Linux 0.11 内核中实现两个新的系统调用：`sys_iam()` 和 `sys_whoami()`。
- 修改内核源码相关文件（如 `sys.h`、`system_call.s`、`unistd.h` 和 `Makefile`）以注册新系统调用。
- 编写用户态测试程序，验证新系统调用的正确性与稳定性。
 
### 核心知识点

#### 一、特权级与保护机制

1. **CPL（Current Privilege Level）** 存储在段寄存器 CS 和 SS 的最低两位，表示当前代码运行的特权级。
2. **DPL（Descriptor Privilege Level）** 存在于 GDT（全局描述符表）中每个段描述符内，表示该段允许被访问的最低特权级。
3. 在 x86 保护模式下，只有当 **CPL ≤ DPL** 时，普通控制转移指令（如 `jmp`、`call`）才能跳转到目标段；否则触发保护异常。
4. 操作系统内核代码段和数据段的 DPL 被初始化为 **0**（最高特权级）。
5. 用户程序运行时 CS 的 CPL 被设为 **3**（最低特权级），因此无法通过普通跳转进入内核。
6. 特权级共分为 4 级（R0～R3），Linux 仅使用 R0（内核态）和 R3（用户态）。

#### 二、GDT 与段机制

7. 操作系统在 `head.s` 初始化阶段建立 GDT 表，并设置内核代码段（DPL=0）、内核数据段（DPL=0）等关键描述符。
8. 段寄存器（如 CS、DS）加载时会自动从 GDT 中读取对应段描述符，并校验特权级。
9. 用户态程序的 CS 值通常为 `0x1B`（二进制 ...00011011），其低两位为 3，即 CPL=3。
10. 内核态 CS 值通常为 `0x08`（二进制 ...00001000），其低两位为 0，即 CPL=0。

#### 三、中断与 IDT

11. **IDT（Interrupt Descriptor Table）** 是中断向量表，存储中断门、陷阱门或任务门描述符。
12. `int 0x80` 是 Linux 系统调用专用的软件中断号。
13. 在内核初始化函数 `sched_init()`（位于 `linux/kernel/sched.c`）中，通过 `set_system_gate(0x80, &system_call)` 设置 0x80 号中断门。
14. `set_system_gate(n, addr)` 宏展开为 `_set_gate(&idt[n], 15, 3, addr)`，其中：
    - 第一个参数 `&idt[n]` 是 **IDT（中断描述符表）中第 `n` 项描述符的内存地址**，指向一个 8 字节的门描述符结构，用于存储中断处理程序的段选择子、偏移地址、DPL 和门类型等信息；
    - 第二个参数 `15` 表示类型为 **陷阱门（Trap Gate）**；
    - 第三个参数 `3` 表示该中断门的 **DPL=3**，允许用户态（CPL=3）调用。
    - 第四个参数 `addr` 是中断/异常发生时 CPU 要跳转执行的处理函数入口地址。
15. 中断门描述符包含：段选择子（如 `0x08`）、偏移地址（如 `&system_call`）、DPL、门类型等字段。

#### 四、系统调用执行流程

16. 用户程序调用 `write(fd, buf, count)` 时，C 库将其转换为一段汇编代码，执行前将：
    - 系统调用号 `__NR_write`（值为 4）写入 **%eax**；
    - 参数依次写入 **%ebx**（fd）、**%ecx**（buf）、**%edx**（count）。
17. 执行 `int 0x80` 后，CPU 自动：
    - 查询 IDT[0x80] 获取中断门；
    - 检查 DPL（=3）≥ CPL（=3），允许进入；
    - 切换栈（若跨特权级），压入用户态 SS、ESP、EFLAGS、CS、EIP；
    - 跳转到段选择子 `0x08`（内核代码段） + 偏移 `system_call` 处执行。
18. 此时 CS = `0x08`，其低两位为 0，故 **CPL 变为 0**，进入内核态。
19. 中断处理程序 `system_call`（位于 `linux/kernel/system_call.s`）首先保存寄存器上下文。
20. 根据 `%eax` 中的系统调用号，通过指令 `call _sys_call_table(,%eax,4)` 跳转到对应的系统调用函数（如 `sys_write`）。
21. `_sys_call_table` 是一个函数指针数组，定义在 `linux/include/linux/sys.h` 中，例如：
    ```c
    fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, ..., sys_write, ... };
    ```
22. 系统调用返回时，通过 `iret` 指令恢复用户态寄存器，CPL 自动恢复为 3。

#### 五、其他细节

28. 并非所有中断都能从用户态进入内核，只有 DPL 被显式设为 3 的中断门才允许。
29. `int 0x80` 是硬件提供的**唯一合法**的用户态主动进入内核态的方式（在 Linux 0.11 中）。
30. 系统调用本质上是通过中断机制实现的受控内核入口，兼具安全性和标准化接口功能。

## 实验步骤

### 一、添加函数说明
- `iam()`
    第一个系统`iam()`，其原型为：
    ```c
    int iam(const char* name);
    ```
    完成的功能是将字符串参数 *name* 的内容拷贝到内核中保存下来。要求 *name* 的长度不能超过 *23* 个字符。返回值是拷贝的字符数。如果 *name* 的字符个数超过了 *23*，则返回 *-1*，并置 *errno* 为 `EINVAL`。
- `whoami()`
    第一个系统`whoami()`，其原型为：
    ```c
    int whoami(char* name, unsigned int size);
    ```
    它将内核中由 `iam()` 保存的名字拷贝到 *name* 指向的用户地址空间中，同时确保不会对 *name* 越界访存（*name* 的大小由 *size* 说明）。返回值是拷贝的字符数。如果 *size* 小于需要的空间，则返回 *-1*，并置 *errno* 为 `EINVAL`。
- 测试程序
    运行添加过新系统调用的 `Linux 0.11`，在其环境下编写两个测试程序 `iam.c` 和 `whoami.c`。最终的运行结果是：
    ```bash
    $ ./iam Junelist

    $ ./whoami

    Junelist
    ```
### 二、添加函数对应系统调用号
在通常情况下，调用系统调用和调用一个普通的自定义函数在代码上并没有什么区别，但调用后发生的事情有很大不同。调用自定义函数是通过 *call* 指令直接跳转到该函数的地址，继续运行。而调用系统调用，是调用系统库中为该系统调用编写的一个接口函数，叫 `API（Application Programming Interface）`。*API* 并不能完成系统调用的真正功能，它要做的是去调用真正的系统调用，过程是：
- 把系统调用的编号存入 *EAX*；
- 把函数参数存入其它通用寄存器；
- 触发 *0x80* 号中断。

所以第一步我们需要做的就是添加对应的系统调用号让其能正确的保存在 *EAX* 寄存器种，通常调用编号的宏保存在 `include/unistd.h`中：
![初始unistd](image\01.png)
从这个未修改的初始文件我们发现原来系统中一共有 *72* 种函数调用号(*0~71*)，下面的定义的函数宏是对 *OS内核* 中系统提供的代码的展开，我们的调用的系统库调用的函数经过不同种类的宏展开成为带有 `int 0x80` 中断的函数，并将系统调用号保存到指定的寄存器以便后续的使用。这里一共有四种函数宏 `_syscall0(int,close,int,fd) `, `_syscall1(type,name,atype,a)`, `_syscall2(type,name,atype,a,btype,b)`, `_syscall3(type,name,atype,a,btype,b,ctype,c)`在这里我们的`iam()`调用是需要有一个参数传入的故其应该展开为`_syscall1(int,iam,const char*,name)`同理`whoami()`应该展开为`_syscall2(int,whoami,char*,name,int,size)`。故我们需要添加系统调用,修改`include/unistd.h`文件:
![修改unistd](image\02.png)

### 三、添加内核函数索引
如果我们仅单单添加了二者的宏我们仅能在用户态面上感受到他们的存在，这个系统调用能在用户态上正确的展开为我们希望的函数，但是通过 `int 0x80` 进入内核后我们无法通过我们定义的宏找到我们内核中对应的函数也就无法执行。那么我们就需要在内核中实现相应的函数，并需要加上一定的机制去实现内核对函数的定位。

为了编写对系统函数的引导，我们需要了解 `int 0x80`的处理过程。这件事情要从整个系统的 *init* 说起，在内核初始化时，主函数调用了 `sched_init()` 初始化函数：
```c
void main(void)
{
//    ……
    time_init();
    sched_init();
    buffer_init(buffer_memory_end);
//    ……
}
```
`sched_init()` 在 `kernel/sched.c` 中定义为:
```c
void sched_init(void)
{
//    ……
    set_system_gate(0x80,&system_call);
}
```
`set_system_gate` 是个宏，在 `include/asm/system.h` 中定义为:
```nasm
#define set_system_gate(n,addr) \
    _set_gate(&idt[n],15,3,addr)
```
在这里给出了只针对于`int 0x80` 处理的代码，在这里`&idt[n]`是处理该中断的表，在这里`system_call`的地址被传向`addr`这个就是处理该中断的函数，所以如果我们想要添加`sys_iam()`和`sys_whoami()`就需要检查`system_call`的生成逻辑保证其能产生我们需要的内核级函数。不过既然我们都说到这里了，不妨继续说完这个调用过程我们接着看`_set_gate()`这个函数，我们应该清楚的认识到直到现在，我们依然处于`CPL = 3`的状态还未真正地到达内核去执行，而这个函数，就是我们真正的进入内核的开始，在初始化处理这个中断的`IDT`时我们把这个表的`DPL`设置为*3*，通过这样我们就有`DPL >= CPL`可以执行这个表中的代码，通过传来的`addr`我们设设置了这个表偏移，而这个表中的段选择符就是把我们的`CS`的后两位置为*0*，这样我们的`CPL`就为*0*了便顺利进入内核态去执行内核态的函数。接下来我们看一下`kernel/system_call.s`：
```nasm

!……
! # 这是系统调用总数。如果增删了系统调用，必须做相应修改
nr_system_calls = 72
!……

.globl system_call
.align 2
system_call:

! # 检查系统调用编号是否在合法范围内
    cmpl \$nr_system_calls-1,%eax
    ja bad_sys_call
    push %ds
    push %es
    push %fs
    pushl %edx
    pushl %ecx

! # push %ebx,%ecx,%edx，是传递给系统调用的参数
    pushl %ebx

! # 让ds, es指向GDT，内核地址空间
    movl $0x10,%edx
    mov %dx,%ds
    mov %dx,%es
    movl $0x17,%edx
! # 让fs指向LDT，用户地址空间
    mov %dx,%fs
    call sys_call_table(,%eax,4)
    pushl %eax
    movl current,%eax
    cmpl $0,state(%eax)
    jne reschedule
    cmpl $0,counter(%eax)
    je reschedule
```
`system_call` 用 `.globl` 修饰为其他函数可见。

`call sys_call_table(,%eax,4)` 之前是一些压栈保护，修改段选择子为内核段，根据汇编寻址方法它实际上是：`call sys_call_table + 4 * %eax`，其中 *eax* 中放的是系统调用号，即 `__NR_xxxxxx`, *4*则是一个标准函数指针的大小。显然，`sys_call_table` 一定是一个函数指针数组的起始地址，它定义在 `include/linux/sys.h` 中：
```nasm
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,...}
```
增加实验要求的系统调用，需要在这个函数表中增加两个函数引用 `sys_iam` 和 `sys_whoami`。当然该函数在 `sys_call_table` 数组中的位置必须和 `__NR_xxxxxx` 的值对应上。所以总结下来我们的修改就是：
- 修改`kernel/system_call.s`的`nr_system_calls`。
- 修改`include/linux/sys.h`的`sys_call_table`并增加额外的函数声明。
![system_call](image\03.png)
![sys](image\04.png)
在这里要注意，一定要注意`sys_iam`和`sys_whoami`的顺序一定要和自己定义的`__NR_iam`和`__NR_whoami`的顺序相同。

不过在这里我们要补充说明，因为我们添加额外的函数文件所以我们必须要更改问哦们的`kernel/MakefileMakefile` 文件，我们需要向其添加我们新写的文件内容，修改前如图：![system_call](image\05.png)
修改后：![sys](image\06.png)
- 第一处：
  ```nasm
  OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o
  ```
  修改为
  ```nasm
  OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o who.o
  ```
  添加了 `who.o`。
- 第二处：
  ```nasm
  ### Dependencies:
  exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
  ```
  修改为
  ```nasm
  ### Dependencies:
  who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
  exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
  ```
  添加了 `who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h`。

### 三、sys函数的实现

终于到实验的最后一步了，我们需要依据修改的`Makefile`的文件在`kernel/who.c`，然后实现两个函数就万事大吉了。先让我们想一下我们我们应该怎么做？首先我们应该在 `sys_iam()`读取用户输入并保存在内核态的某个区域，`sys_whoami()`要将`sys_iam()`读取的用户数据重新保存在`name`指向的位置并返回用户输入的长度。

那在这里我们便遇到了第一个难题：`在用户态和核心态之间传递数据？`指针参数传递的是应用程序所在地址空间的逻辑地址，在内核中如果直接访问这个地址，访问到的是内核空间中的数据，不会是用户空间的。所以这里还需要一点儿特殊工作，才能在内核中从用户空间得到数据。要实现的两个系统调用参数中都有字符串指针，非常像 `open(char *filename, ……)`，所以我们看一下 `open()` 系统调用是如何处理的:
```c
int open(const char * filename, int flag, ...)
{
//    ……
    __asm__("int $0x80"
            :"=a" (res)
            :"0" (__NR_open),"b" (filename),"c" (flag),
            "d" (va_arg(arg,int)));
//    ……
}
```
可以看出，系统调用是用 `eax、ebx、ecx、edx` 寄存器来传递参数的。
- 其中 `eax` 传递了系统调用号，而 `ebx、ecx、edx` 是用来传递函数的参数的
- `ebx` 对应第一个参数，`ecx` 对应第二个参数，依此类推。
  
如 `open` 所传递的文件名指针是由 `ebx` 传递的，也即进入内核后，通过 `ebx` 取出文件名字符串。`open` 的 `ebx` 指向的数据在用户空间，而当前执行的是内核空间的代码，如何在用户态和核心态之间传递数据？接下来我们继续看看 open 的处理：
```nasm
system_call: //所有的系统调用都从system_call开始
!    ……
    pushl %edx
    pushl %ecx
    pushl %ebx                # push %ebx,%ecx,%edx，这是传递给系统调用的参数
    movl $0x10,%edx            # 让ds,es指向GDT，指向核心地址空间
    mov %dx,%ds
    mov %dx,%es
    movl $0x17,%edx            # 让fs指向的是LDT，指向用户地址空间
    mov %dx,%fs
    call sys_call_table(,%eax,4)    # 即call sys_open
```
由上面的代码可以看出，获取用户地址空间（用户数据段）中的数据依靠的就是段寄存器 *fs*，下面该转到 `sys_open` 执行了，在 `fs/open.c` 文件中：
```c
int sys_open(const char * filename,int flag,int mode)  //filename这些参数从哪里来？
/*是否记得上面的pushl %edx,    pushl %ecx,    pushl %ebx？
  实际上一个C语言函数调用另一个C语言函数时，编译时就是将要
  传递的参数压入栈中（第一个参数最后压，…），然后call …，
  所以汇编程序调用C函数时，需要自己编写这些参数压栈的代码…*/
{
    ……
    if ((i=open_namei(filename,flag,mode,&inode))<0) {
        ……
    }
    ……
}
```
它将参数传给了 `open_namei()`。再沿着 `open_namei()` 继续查找，文件名先后又被传给`dir_namei()、get_dir()`。在 `get_dir()` 中可以看到：
```c
static struct m_inode * get_dir(const char * pathname)
{
    ……
    if ((c=get_fs_byte(pathname))=='/') {
        ……
    }
    ……
}
```
处理方法就很显然了：用 `get_fs_byte()` 获得一个字节的用户空间中的数据。所以，在实现 `iam()` 时，调用 `get_fs_byte()` 即可。但如何实现 `whoami()` 呢？即如何实现从核心态拷贝数据到用心态内存空间中呢？猜一猜，是否有 `put_fs_byte()`？有！看一看 `include/asm/segment.h` ：
```c
extern inline unsigned char get_fs_byte(const char * addr)
{
    unsigned register char _v;
    __asm__ ("movb %%fs:%1,%0":"=r" (_v):"m" (*addr));
    return _v;
}
```
```c
extern inline void put_fs_byte(char val,char *addr)
{
    __asm__ ("movb %0,%%fs:%1"::"r" (val),"m" (*addr));
}
```
事实上他俩以及所有 `put_fs_xxx()` 和 `get_fs_xxx()` 都是用户空间和内核空间之间的桥梁,这点非常重要。

接下来就是我们编写的`kernel/who.c`：
```c
#include<string.h>
#include<asm/segment.h>  // get_fs_byte, put_fs_byte
#include<errno.h>

char str_pos[24];
int sys_iam(const char* name)
{
    char c ;
    int i = 0;
    while((c = get_fs_byte(name+i)) != '\0')
    {
        str_pos[i] = c;
        ++i;
    }

    if(i > 23)
    {
        errno = EINVAL;
        return -1;
    }
    printk("The User Input is:  %s\n",str_pos );  
    return i;
}

int sys_whoami(char* name, unsigned int size)
{
    if(size<strlen(str_pos))
    {
        errno = EINVAL;
        return -1;
    }
    int ans = 0;
    char c;
    while((c = str_pos[ans] )!='\0')
    {
        put_fs_byte(c,name++);
        ++ans;
    }
    return ans;
}
```
在这里我要简短的补充一个小知识，`printk`函数就是内核态的 `printf` 函数，用它来帮助我们在内核态调试很方便，那我们能否用 `printf`来替代呢？很显然不能，因为在内核态中我们无法处理用户态的函数，我们的内核中没有`printf`这个函数。看似能用的 `strlen` 等字符串函数，实际上是**内核自己重新实现的版本**，并非用户态的标准库函数。Linux 内核在 `lib/string.c` 中提供了专门为内核环境优化的字符串处理函数，这些函数不依赖任何用户态库，针对内核内存管理进行优化而且确保在内核异常情况下的可靠性。
![return](image\07.png)如果一切正常，我们会通过编译，如果出现了某些错误，请您仔细检查，但不要会心，错误至少证明了我们`Makefile`修改成功了。接下来让我们编写在`Linux 0.11`这个小系统编译的两个测试文件并上载到上面吧！
- iam.c
  ```c
  #define __LIBRARY__   // 定义了这个宏，unistd.h中的一个条件编译块才会编译
  #include <unistd.h>
  #include <errno.h>
  _syscall1(int, iam, const char*, name);


  int main(int argc, char* argv[])
  {
      iam(argv[1]);
      return 0;
  }
  ```
- whoami.c
  ```c
  #define __LIBRARY__
  #include <unistd.h>
  #include <errno.h>
  #include <stdio.h>

  _syscall2(int, whoami, char*, name, unsigned int, size);

  int main()
  {   
      char username[25] = {0};
      whoami(username, 23);
      printf("username: %s\n", username);
      return 0;
  }
  ```
我们首先回到`oslab`目录下依次执行：
```bash
$ sudo ./mount-hdc 
$ cd hdc/usr/root
$ vim iam.c
$ vim whoami.c
```
注意在`iam.c,whoami.c`程序内的头文件`<unistd.h>`是标准头文件，是由*GCC*编译器一同安装的，它们通常随着*GCC*一起打包并分发，通常位于`/usr/include`目录下，而不是在之前修过的源码树下的`include/unistd.h`, 因此我们要转入`hdc/usr/include`下修改`<unistd.h>`，加入两个宏`__NR_iam,__NR_whoami `编译。
当我们修改完了之后再回退到`oslab`目录下执行`sudo umount hdc`取消系统盘的挂载，之后进入系统编译对应文件如果一切都没错误那么会出现：![final](image\08.png)至此，试验结束。
