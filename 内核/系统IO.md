[TOC]

# 系统I/O

## 一、I/O的概念

> 输入/输出(I/O)是在内存和外设之间复制数据的过程。输入操作是从外设复制数据到内存，输出操作是从内存复制数据到外设。
>

所有语言提供的I/O库，如C语言的`printf`、`scanf`，C++重载的`>>`、`<<`，都被称为**标准I/O**接口，它们都是在操作系统提供的**系统级I/O**接口的基础上实现的。

### Linux下一切皆文件

Linux系统秉持着=="一切皆文件"==的原则，将所有的I/O设备，如磁盘、键盘、网络等，都模型化为统一的**"文件"**。

其最大的好处在于：允许用户使用同一套系统I/O接口对这些设备进行输入/输出操作。

 

## 二、系统级I/O接口

### 1、打开文件

`int open(const char *pathname, int flags, .../*mode_t mode*/)`

- `pathname`：目标文件的绝对路径或相对当前工作目录的路径
- `flags`：打开方式，包括：

1. `O_RDONLY`：只读

2. `O_WRONLY`：只写

3. `O_RDWR `：可读可写

4. `O_APPEND`：追加，必须和`O_WRONLY`搭配使用，即`O_APPEND | O_WRONLY`

5. `O_CREAT`：如果目标文件不存在，则以pathname为路径创建该文件

> 注：flags可以是几个标志按位或，比如O_CREATE | O_WRONLY表示写文件，如果该文件不存在，则先创建。

- `mode`：可变参数列表中的参数，如果需要创建文件，则需要传该参数(8进制)作为新文件的默认权限，最终权限由`“默认权限 & ~umask”`决定，其中`umask`为文件权限掩码
- RetVal：返回打开文件的文件描述符，用户通过该文件描述符进行I/O操作

### 2、读文件

`ssize_t read(int fd, void *buf, size_t count)`

- `fd`：目标文件描述符
- `buf`：输出型参数，用作存储缓冲区
- `count`：最多读取的字节数
- RetVal：一个`ssize_t`类型的整数(本质为`long`类型)。如果成功，则返回本次实际读取到的字节数，0表示遇到EOF，失败则返回-1


### 3、写文件

`ssize_t write(int fd, const void *buf, size_t count)`

- `fd`：目标文件描述符
- `buf`：输入型参数，写入内容的存储缓冲区
- ` count`：要写入的字节数，一般为`sizeof(buf)`
- RetVal：一个`ssize_t`类型的整数(本质为long类型)，如果成功，则返回本次实际写入的字节数，失败则返回-1

### 4、关闭文件

`int close(int fd)`

将文件描述符为`fd`的文件关闭，成功则返回0，失败则返回-1。

注：所谓关闭，就是将当前占用`fd`的文件相关结构体释放，之后该`fd`还可被分配给新打开的文件。



## 三、文件描述符(file descriptor)

### 1、什么是文件描述符

文件描述符，简称`fd`，本质是一个**非负整数**，用户可以使用它来进行对应文件的I/O操作。

Linux内核为了记录每个进程打开的哪些文件，在进程控制块**task_struct**中保存了一个维护已打开文件的结构体**files_struct**，该结构体内部维护了一个文件记录表**`fd_array`**用来保存文件的具体信息。而所谓的`fd`，本质上就是这个文件记录表的数组下标，内核通过`fd`就可以索引到具体的文件。



![image-20220806202932150](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062029192.png) 

### 2、文件操作方法

![image-20220806202906240](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062029436.png) 

**`fd_array`**数组中存储了file结构体，该结构体包含了各种文件相关信息。其中，**f_op**是一个`file_operations`结构体，该结构体维护了操作该文件的各种函数指针，如`read`、`write`等。

> 由此可以看出：
>
> Linux下将各种I/O设备都模型化为文件系统的一个结构体，且根据不同类型的文件，操作系统会为它注册不同的操作方法。

### 3、文件描述符分配规则

Linux下每个进程都会默认打开三个“文件”：标准输入、标准输出和标准错误，它们的文件描述符分别为0、1、2。

之后再打开的文件，它的文件描述符是<font color=red>**当前未被占用的最小的fd**</font>。

#### C语言下的三大标准文件

这三个默认文件在C语言中分别对应FILE*类型的文件指针：`stdin、stdout、stderr`。

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062003161.jpg) 

C语言的`FILE`结构体原型是`_IO_FILE`：

<img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062003163.jpg" alt="img" style="zoom:150%;" /> 

其中的`_fileno`保存文件描述符，其它的各类char*类型指针对应C语言维护的**用户级缓冲区**等信息。

### 4、文件描述符的数量限制

Linux下可以通过`ulimit -n`，即`ulimit -a中的open files选项`查看**每个进程**可以同时打开的最大描述符数量(一般是1024)。

通过`cat /proc/sys/fs/file-max`查看**当前系统**可以同时打开的最大文件描述符数量。

 

## 四、用户缓冲区与内核缓冲区

> 用户级缓冲区由高级语言提供，是进程运行时用户空间的一段虚拟内存，如C语言的用户级缓冲区由FILE结构体维护。当用户向“文件”中读写时，优先使用这部分缓冲区的内容。
>

> 内核级缓冲区是由操作系统在内核空间为进程打开的文件所维护的，每个文件都拥有自己的缓冲区。
>

为了方便理解，以读写为例：

1、当用户使用标准I/O接口进行==“读”操作==时，首先操作系统会查看当前的用户缓冲区是否有可读数据，如果没有，则查看内核缓冲区是否有数据，如果还没有，则先将数据从外部I/O设备拷贝至内核缓冲区，再将内核缓冲区的数据拷贝至用户缓冲区；

![image-20220806201829915](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208131017467.png)

2、当用户使用标准I/O接口进行==“写”操作==时，首先将数据拷贝至语言层提供的用户缓冲区。当满足语言规定的刷新策略时(比如行缓冲、全缓冲)，操作系统会将用户缓冲区的数据拷贝至内核缓冲区，之后再根据内核提供的刷新策略(比如积累到一定量)将数据写入磁盘、网卡等外部I/O设备。

 ![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208131023483.png)

### 1、缓冲区刷新策略

- 全缓冲


当读写的数据填满缓冲区时才进行对应的I/O操作，典型代表是对磁盘文件的读写。

- 行缓冲


当输入和输出遇到换行符时才进行I/O操作，典型代表是键盘、显示器。

- 不缓冲


不设立缓冲区,典型代表是`stderr`，方便错误信息快速地显示出来。

- 其它刷新策略

进程结束后自动刷新、强制刷新(`fflush`接口).

### 2、子进程与缓冲区

```c
printf("printf()\n");
fputs("fputs()\n", stdout);

char* buffer = "write()\n";
write(1, buffer, strlen(buffer));

fork();
```

当==直接在命令行运行该程序==时，输出结果为：

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062003832.jpg) 

当==把输出结果重定向至文件中==时，文件内容为：

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062003857.jpg) 

可以看到，<font color=red>文件中标准I/O接口`printf`和`fputs`分别输出了两次，而系统调用接口write仅输出一次</font>，原因在于：

1. 进程会创建父进程的虚拟地址空间副本，那么用户空间的用户缓冲区也理所应当地被子进程拿到。由于**写时拷贝**的存在，父子进程的<font color=red>**用户缓冲区内容相同但相互独立**</font>，因此会被刷新两次；
2. `write`属于系统调用接口，而系统调用接口会<font color=red>直接将数据拷贝至内核缓冲区，不存在用户级缓冲区</font>；
3. 使用`./test`命令时，`printf`和`fputs`会向显示器文件写入数据。而显示器文件的刷新策略是行缓冲，因此在遇到'\n'时，用户缓冲区的数据被立即刷新至内核缓冲区，并由内核决策何时将其写入显示器文件。

4. 使用重定向`>`时，`printf`和`fputs`会向普通文件中写入数据。而普通文件的刷新策略是全缓冲，只有在用户缓冲区满时或进程结束才会将数据刷新至内核缓冲区，并由内核决策何时将将其写入普通文件。


### 3、缓冲区的意义

- 对于内核缓冲区：
  1. 由于内核是各种硬件驱动的管理者，因此用户无法越过内核而直接读写外设数据。
  2. 外设的读写速度较慢，因此设立内核缓冲区，**避免了与外设的频繁I/O**，提高了系统的性能。
  3. 从缓冲区直接读数据提高了读操作的效率。
  4. 将数据直接写入缓冲区，至于什么时候刷新到磁盘由操作系统决定**（延迟写）**，提高了写操作的效率。

- 对于用户缓冲区：

  向内核缓冲区读写数据的标准I/O接口**本质上使用了read()、write()这些系统调用接口**。每当调用系统接口时，CPU都要从用户态切换至内核态，而这种切换会有时间消耗。当I/O较为频繁时，CPU状态切换的消耗就会相应叠加。因而设立用户缓冲区，减少因为系统调用而进行状态切换的次数。

### 4、标准I/O与系统I/O对比

1. 标准I/O有用户级缓冲区，而系统I/O没有，因此标准I/O可以先将数据写入用户缓冲区，当缓冲区满再切换至内核态将其拷贝至内核缓冲区；而系统I/O每一次写都要切换至内核态，再向内核缓冲区写数据。因此**标准I/O的效率更高**。
1. 有些情况下必须使用系统I/O，比如向管道文件中读写、套接字编程。

### 5、零拷贝

> 零拷贝是一种避免将数据从一个内存缓冲区拷贝至另一内存缓冲区的技术。

#### I. 零拷贝原理

对于传统IO，如果想要在网络上发送一个文件，则需要：

1. 陷入内核，将文件从硬盘拷贝至内核缓冲区，再将数据从内核拷贝至用户态；
2. 陷入内核，将数据从用户态拷贝至内核的socket发送缓冲区；

很明显，第1步中从内核拷贝到用户态的消耗是完全没必要的。Linux下的sendfile与mmap可以做到省略该步骤。

- 调用sendfile时：

  1. CPU陷入内核并将文件数据拷贝至内核缓冲区。
  2. CPU将数据从内核缓冲区拷贝至socket发送缓冲区。

  **减少了拷贝与陷入CPU状态切换的次数**，因此效率相比传统IO更高。

- 调用mmap时：由于直接使用内存映射技术，因此内核与用户都能看到内存中的同一份文件数据，无需数据拷贝。

#### II. 零拷贝的好处

1. 减少用户态和内核态之间数据拷贝，提升数据传输效率。
2. 减少用户态和内核态之间的上下文切换的消耗。

## 五、重定向

> 为了方便用户将命令的输出结果保存到文件而不是直接在显示器上打印，bash提供了两个重定向操作符`>`和`>>`；
>

- `命令 > 文件名 `将对应文件内容清空，然后将命令的运行结果写入
- `命令 >> 文件名`在对应文件已有内容后追加

### 1、系统调用dup2()

`int dup2(int oldfd, int newfd) // 失败返回-1`

该函数本质上修改了当前进程控制块task_struct的文件记录表`fd_array`：先将原先`newfd`对应的文件结构体被删除，再将`fd_array[oldfd]`复制到`fd_array[newfd]`，使得`oldfd`和`newfd`对应同一个文件

注：<font color=red>一个文件可以有不止一个fd与之对应</font>。

### 2、重定向操作符的实现原理

> 每个文件描述符都对应一个文件的描述信息用于操作文件。而重定向就是在不改变所操作的文件描述符的情况下，通过改变描述符对应的文件描述信息进而实现改变所操作的文件

进程创建伊始会打开三个默认文件，其中**fd=1**的文件是显示器的**标准输出文件**。

当调用`dup2(fd, 1)`时，标准输出文件的1号文件描述符对应的文件结构体被描述符为fd的新文件结构体覆盖，而`printf`这些I/O接口<font color=red>默认会将结果输出到1号文件</font>，因此程序的运行结果就被写入到fd对应的文件了。

#### I.模拟实现

```C
int fd = open("f.txt", O_CREAT | O_WRONLY, 0644);
dup2(fd, 1);
printf("dup2()\n");
```

或者根据dup2()的原理，将代码改写成：

```C
close(1);
// 注：将1号文件close后，新打开文件的fd自然就是1，那么printf就会将结果写入该文件了。 
int fd = open("f.txt", O_CREAT | O_WRONLY, 0644);
printf( "dup2()\n");
```

#### II.运行结果

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208062003973.jpg) 









