# 0、写在前面

为了能将本文的内容讲解清楚，我觉得很有必要简单文章中出现几个标准及相关历史。尽管计算机的发展历史还不到一个世纪，但很多问题如果不了解相关历史背景很难一窥全貌。

## 0.1、UNIX相关历史

1970年，在那个计算机仍处于混沌状态的年代，[AT＆T](https://en.wikipedia.org/wiki/AT%26T_Corporation)公司[贝尔实验室](https://en.wikipedia.org/wiki/Bell_Labs)的两位研究员[Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson)和[Dennis Ritchie](https://en.wikipedia.org/wiki/Dennis_Ritchie)刚经历[Multics](https://en.wikipedia.org/wiki/Multics)大型机操作系统研发失败的遭遇。Ken叔和Dennis退出后并没有因此感到沮丧，而是在[PDP-11](https://en.wikipedia.org/wiki/PDP-11)小型机上，继续了他们未竟之功业，最终这两位开天辟地的程序员用[汇编语言](https://en.wikipedia.org/wiki/Assembly_language)完成了[Unix](https://en.wikipedia.org/wiki/Unix)第一版。

> 基于1970年1月1日的[Unix时间](https://en.wikipedia.org/wiki/Unix_time)也被作为后来各种编程语言[系统时间](https://en.wikipedia.org/wiki/System_time)。

实验室的其他同事都觉Unix甚好，为了方便操作系统移植到其他机器，不能再使用汇编语言这种[机器指令](https://en.wikipedia.org/wiki/Machine_code)助记符了。经过几个版本的迭代，从B语言演化到C语言，终于1973年使用C语言重写的Unix第四版发布了。后续在一些同事的帮助下，Unix变得越来越受人欢迎，有几个同事还专门为Unix写了新的[Shell外壳程序](https://en.wikipedia.org/wiki/Unix_shell)，其中包括[Steve Bourne](https://en.wikipedia.org/wiki/Stephen_R._Bourne)为Unix写的[Bourne shell](https://en.wikipedia.org/wiki/Bourne_shell)，B Shell在1976年随Unix第七版发布并作为默认的Shell程序。另外，为了将C语言推广出去，Dennis和同事[Brian Kernighan](https://en.wikipedia.org/wiki/Brian_Kernighan)在1978年出版了《[*The C Programming Language*](https://en.wikipedia.org/wiki/C_%28programming_language%29)》这部伟大著作，这个C语言版本也被称为[*K&R C*](https://en.wikipedia.org/wiki/C_%28programming_language%29#K&R_C)，也被认为是C语言第一个事实上的标准。

由于Unix倍受欢迎，Unix的源码由早期授权给大学学术研究，后流传到了各个商业公司，从而衍生出了越来越多的分支。其中比较有名的就是加州伯克利分校的Unix分支——[BSD(Berkeley Software Distribution)](https://en.wikipedia.org/wiki/Berkeley_Software_Distribution)，由他逐渐产生出了OpenBSD，FreeBSD，NetBSD等各分支。另外一个重要分支是Unix第一个商业授权版[SystemV](https://en.wikipedia.org/wiki/UNIX_System_V)(V是罗马数字5,也就是贝尔实验室Unix第五版)，由他逐渐产生了各商业公司的版本，如[IBM](https://en.wikipedia.org/wiki/IBM)的[AIX](https://en.wikipedia.org/wiki/AIX_operating_system)，[HP](https://en.wikipedia.org/wiki/Hewlett-Packard)的[HP-UX](https://en.wikipedia.org/wiki/HP-UX)，[Sun](https://en.wikipedia.org/wiki/Sun_Microsystems)的[Solaris](https://en.wikipedia.org/wiki/Solaris_(operating_system))。除了这几个主流分支外，还有很多其他小分支。比如我们目前用的[MacOS](https://en.wikipedia.org/wiki/MacOS)就是众多分支中的一个，甚至于后来的[微软](https://en.wikipedia.org/wiki/Microsoft)也维护了自己的Unix版本[Xenix](https://en.wikipedia.org/wiki/Xenix)。

![Unix发展历史](https://upload.wikimedia.org/wikipedia/commons/7/77/Unix_history-simple.svg)

Unix的开放导致了各个公司都可以拓展操作系统接口，甚至对C语言进行各种改进和扩展。为了能让程序能方便在各个系统间移植，标准化进程迫在眉睫。

C语言方面，1989年由[美国国家标准委员会(ANSI)](https://en.wikipedia.org/wiki/American_National_Standards_Institute)完成制定了第一个国家标准C89，随后[ANSI C](https://en.wikipedia.org/wiki/ANSI_C)标准便被[国际标准化组织(ISO)](https://en.wikipedia.org/wiki/International_Organization_for_Standardization)采纳，后续根据计算机的发展需求每隔几年便会发布新标准。标准C中定义了C语言语法与编程最常用的[库函数](https://en.wikipedia.org/wiki/C_standard_library)，目前BSDlibc，glibc，vc++都有相关库函数实现。

Unix系统接口方面，[IEEE计算机学会](https://en.wikipedia.org/wiki/IEEE_Computer_Society)制定了[POSIX标准](https://en.wikipedia.org/wiki/POSIX)(Portable Operating System Interface，可移植操作系统接口)。POSIX标准定义了Unix操作系统的应用编程接口，如线程、信号量、内存分配、I/O等。随着标准的发展，POSIX还囊括了Shell以及[用户级应用程序](http://pubs.opengroup.org/onlinepubs/9699919799/idx/utilities.html)(如awk、echo、ed)的相关标准。

由于IEEE对POSIX规范收取大量费用，有些公司对规范置若罔闻，这很明显有悖POSIX标准的初衷。于是几个重要的Unix供应商发起了Common API Spec并组建了[COSE](https://en.wikipedia.org/wiki/Common_Open_Software_Environment)联盟，后更名为[X/Open](https://en.wikipedia.org/wiki/X/Open)，1990年联盟成员已经扩大到21个，1996年X/Open与[OSF基金会](https://en.wikipedia.org/wiki/Open_Software_Foundation)合并成立了如今的[OpenGroup](https://en.wikipedia.org/wiki/The_Open_Group)。OpenGroup发布了[Single UNIX Specification](https://en.wikipedia.org/wiki/Single_UNIX_Specification)。从[官网](http://www.unix.org/what_is_unix/single_unix_specification.html)可以看到，SUS包含了规范、产品、技术、商标四个元素，只有符合SUS相关标准的系统才会被认证为Unix系统，从而使用Unix的商标，否则被称为[类Unix系统(Unix-like)](https://en.wikipedia.org/wiki/Unix-like)。我们可以简单的认为SUS是POSIX标准的超集。

![SUS](http://ww1.sinaimg.cn/large/bda5cd74ly1g0kaawvqkag20bh0663ye.gif)

> MacOS已被认证为Unix操作系统，而Linux仍为类Unix系统。

## 0.2、内核系统调用与库函数调用

狭义的操作系统通常只包括[系统内核](https://en.wikipedia.org/wiki/Kernel_%28operating_system%29)，比如[Linux Kernel](https://www.kernel.org/)、[Windows Kernel](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/_kernel/)。

而[系统调用](https://en.wikipedia.org/wiki/System_call)就是内核暴露的服务接口，这些服务包括进程/线程管理，内存管理，I/O，设备管理等内核层面的功能。

[系统调用通常不会直接被调用](http://man7.org/linux/man-pages/man2/syscalls.2.html)，会有一层薄薄的包装层，其中一方面原因是为了兼容并实现POSIX标准定义的系统调用API，在Linux中glibc实现了这个包装层。如果想要[显式的进行系统调用](https://www.gnu.org/software/libc/manual/html_node/System-Calls.html)，在glibc库中可以使用[syscall函数](http://man7.org/linux/man-pages/man2/syscall.2.html)。

> 到目前(2019年)为止[posix1003.1](http://pubs.opengroup.org/onlinepubs/9699919799/)定义了1191个系统接口函数，[这里](http://pubs.opengroup.org/onlinepubs/9699919799/idx/functions.html)有posix系统接口的列表。

我们可以认为这些包装层函数就是系统内核提供给我们的内核API。

这些内核API除了实现了POSIX标准中定义的操作系统功能外，还有POSIX未定义的操作系统特有API。比如后面要介绍到的Linux中的epoll函数，和BSD系的kqueue函数。

POSIX中除了操作系统接口相关的函数外，还包括[C标准库](https://en.wikipedia.org/wiki/C_standard_library)定义的函数。换句话说POSIX就是C标准库的超集。

C标准库中的一些函数只是对操作系统接口进行了封装，最典型的就是标准I/O，它是基于操作系统的无缓冲I/O封装了一层。C语言库函数中还有一些与操作系统无关的工具函数，如字符串处理、复数、伪随机数生成器等，这些函数不会与操作系统接口交互。

在Windows中系统内核API都暴露在[kernel.dll文件](https://en.wikipedia.org/wiki/Microsoft_Windows_library_files#KERNEL32.DLL)中，使用Windows提供的内核API可以创建相应的[Windows内核对象](https://docs.microsoft.com/en-us/windows/desktop/SysInfo/kernel-objects)。但Windows API并没有遵循POSIX规范，所以就有第三方应用[基于Windows API开发了兼容POSIX标准的兼容层](https://en.wikipedia.org/wiki/POSIX#POSIX_for_Microsoft_Windows)，可以让Unix/Linux上的软件尽可能不用修改代码就可以在Windows上重新编译，其中比较著名的就是[Cygwin](https://en.wikipedia.org/wiki/Cygwin)和[MinGW](https://en.wikipedia.org/wiki/MinGW)。在Windows上能使用git也是得益于这个兼容层。尽管Windows没有遵循POSIX规范，但它提[C标准库的实现](https://docs.microsoft.com/en-us/cpp/c-runtime-library/c-run-time-library-reference?view=vs-2017)，除此之外还对C库中的一些函数进行了拓展，比如对臭名昭著的字符串缓冲区溢出提供了[安全检查](https://docs.microsoft.com/en-us/cpp/c-runtime-library/security-features-in-the-crt?view=vs-2017)。

![系统调用与库函数调用](http://ww1.sinaimg.cn/large/bda5cd74ly1g0poimaylyj20p20nm7bw.jpg)

这里有Windows和POSIX常见内核API的简单对照表

|          | Windows                                                      | POSIX                                                        |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 进程控制 | [CreateProcess()](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessa)<br />[ExitProcess()](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-exitprocess)<br />[WaitForSingleObject()](https://docs.microsoft.com/en-us/windows/desktop/api/synchapi/nf-synchapi-waitforsingleobject) | [fork()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html)<br />[exit()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/exit.html)<br />[wait()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/wait.html) |
| 文件操作 | [CreateFile()](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-createfilea)<br />[ReadFile()](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-readfile)<br />[WriteFile()](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-writefile)<br />[CloseHandle()](https://docs.microsoft.com/en-us/windows/desktop/api/handleapi/nf-handleapi-closehandle) | [open()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html)<br />[read()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/read.html)<br />[write()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html)<br />[close()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html) |
| 设备操作 | [SetConsoleMode()](https://docs.microsoft.com/en-us/windows/console/setconsolemode)<br />[ReadConsole()](https://docs.microsoft.com/en-us/windows/console/readconsole)<br />[WriteConsole()](https://docs.microsoft.com/en-us/windows/console/writeconsole) | [ioctl()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/ioctl.html)<br />[read()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/read.html)<br />[write()](http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html) |
| 内存管理 | [HeapAlloc()](https://docs.microsoft.com/zh-cn/windows/desktop/api/heapapi/nf-heapapi-heapalloc)<br />[HeapFree](<https://docs.microsoft.com/en-us/windows/desktop/api/HeapApi/nf-heapapi-heapfree>)<br />and [so on](<https://docs.microsoft.com/en-us/windows/desktop/memory/comparing-memory-allocation-methods>) | [brk()](<https://pubs.opengroup.org/onlinepubs/7908799/xsh/brk.html>)<br />[sbrk()](<https://pubs.opengroup.org/onlinepubs/7908799/xsh/sbrk.html>)<br />[malloc()](<http://pubs.opengroup.org/onlinepubs/9699919799/functions/malloc.html>)<br />[free()](<http://pubs.opengroup.org/onlinepubs/9699919799/functions/free.html>)<br /> |

我们对操作系统API有了最基本的认识，下面就进入这篇文章的主题

# 1、阻塞式IO

和数据处理相比，计算机的IO速度非常慢。因为CPU发出读写IO的指令后，IO设备可能会包括机械硬盘物理寻道(磁盘IO)或者调制解调器将电信号转换成模拟信号并在信道内传输(网络IO)的过程。这也意味着处理器会花宝贵的时间去等IO操作完成，当然操作系统不允许这么暴殄天物，聪明的操作系统会帮我们自动切换到其他进程或线程去处理其他计算任务，但无可厚非地需要浪费切换操作所消耗的时间。

对于磁盘IO来说阻塞式未尝不可，因为除了磁盘阵列和SSD，普通的机械硬盘由于机械寻道导致难以并发读写。

而对于服务器使用阻塞式网络IO，那只能靠一个线程处理一个用户的网络连接，为了能处理更多的用户请求，只有创建更多的线程。这也将导致几个问题：

1、通常操作系统对线程数有限制。

2、线程执行栈消耗内存，更多的线程意味着需要更多的内存，线程数也受限于内存大小。

3、过多的线程，意味着更多的上下文切换，CPU浪费的时间更多。

创建这么多线程，都只是为了等待IO完成，简直就是对资源的浪费。

> 目前C标准库中所有的[IO操作](https://en.cppreference.com/w/c/io)以及POSIX的[socket api](<https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/sys_socket.h.html>)都是同步阻塞式的。
>
> [这里](https://github.com/holmofy/echo-server/blob/master/tcp-echo-server.c)有一个单线程的echo-server的实现，单线程的缺点是每次只能为一个客户端服务。
>
> [这里](https://github.com/holmofy/echo-server/blob/master/tcp-echo-server-multithread.c)有一个多线程的echo-server的实现，多线程的缺点是每个客户端连接需要创建一个线程，线程过多会浪费系统资源。

#2、IO多路复用

[多路复用(Multiplex)](<https://en.wikipedia.org/wiki/Multiplexing>)这个词来自于计算机网络，是一种通过共享信道将多个模拟信号合成一个信号的方法。

![时分多路复用原理图](http://ww1.sinaimg.cn/large/bda5cd74ly1g14qlfflpzg20c703nn4t.gif)

而IO多路复用指的就是让一个线程专门负责监听每个网络连接的状态，一旦有数据可以读取就通知工作线程进行处理。

这就好比到餐厅吃饭，传统的阻塞式IO就是一个服务员服务一桌客人，从你进餐厅到结账全程一对一的VIP服务。对于餐厅老板来说这个成本太高了，1000桌就得聘请1000个服务员。而且服务员在为客人服务的过程中并不是一直都忙着，客人点完菜，上完菜，吃着的这段时间，服务员就闲下来了，可是这个服务员还是被这桌客人占用着，不能为别的客人服务。

IO多路复用让一个服务员作为前台专门负责收集客人的需求，登记下来，比如有客人进来了、客人点菜了，客人要结帐了，都先记录下来按顺序排好。每个服务员到这里领一个需求，比如点菜，就拿着菜单帮客人点菜去了。点好菜以后，服务员马上回来，领取下一个需求，继续为别人客人服务去了。这种方式服务质量就不如一对一的服务了，当客人数据很多的时候可能需要等待。但好处也很明显，由于在客人正吃饭着的时候服务员不用闲着了，服务员这个时间内可以为其他客人服务了，原来10个服务员最多同时为10桌客人服务，现在可能为50桌，60客人服务了。

> 上面这个比喻引用自CSDN博主[zhouhl_cn](https://blog.csdn.net/zhouhl_cn/article/details/6568119)的一篇文章

## 2.1、 poll、select

![select模型](http://ww1.sinaimg.cn/large/bda5cd74ly1g3i5qookozg20d70cz0t1.gif)

<https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/poll.h.html>



<https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/sys_select.h.html>

## 2.2、epoll、kqueue

<http://man7.org/linux/man-pages/man7/epoll.7.html>

<https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2>

# 3、异步IO

[AIO](http://man7.org/linux/man-pages/man7/aio.7.html)

<https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/aio.h.html>

[IOCP](https://docs.microsoft.com/zh-cn/windows/desktop/FileIO/i-o-completion-ports)

https://docs.microsoft.com/en-us/windows/desktop/WinSock/windows-sockets-start-page-2

# 4、Java NIO

![IO Channel](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLSCv8pCjBpK4I2KfDJ4bCoabrgYn9nPGDByeiGPT5aWvEJYm1iZFpqh5hTqyioK18O-h7hYiuDJKRmr8eGy6cHbSNcwDH561tDwekX6eSte2OWQaSuIlYJ4Staw0YT4di0UAGcfS2Z5q0)





参考资料：

《[Windows核心编程](https://book.douban.com/subject/3235659/)》

《[Linux/UNIX系统编程手册](https://book.douban.com/subject/25809330/)》

《[UNIX环境高级编程](https://book.douban.com/subject/25900403/)》

https://docs.microsoft.com/zh-cn/windows/desktop/FileIO/i-o-concepts

https://docs.microsoft.com/zh-cn/windows/desktop/FileIO/synchronous-and-asynchronous-i-o

http://www.drdobbs.com/open-source/io-multiplexing-scalable-socket-servers/184405553

https://en.wikipedia.org/wiki/Non-blocking_I/O_%28Java%29

https://en.wikipedia.org/wiki/Asynchronous_I/O

https://tinyclouds.org/iocp-links.html

https://en.wikipedia.org/wiki/Kqueue

https://en.wikipedia.org/wiki/C10k_problem

https://stackoverflow.com/questions/39802643/java-async-in-servlet-3-0-vs-nio-in-servlet-3-1

https://www.ibm.com/support/knowledgecenter/ssw_ibm_i_74/rzab6/xnonblock.htm