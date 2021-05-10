---
title: Win32开发小回忆
date: 2018-04-13 23:31
tags:
categories: Windows
---

这两天阿瘦找我给他的一个程序写个界面，听说是要参加啥三创比赛(都大四老狗了，汗)，然后问要用什么语言——C/C++，Windows平台的。他之前没怎么接触过C++方面的界面开发，然后我就开始了一波Windows教学，顺便自己也回忆回忆(大一大二玩了一年多，之后几乎就没碰过)。

# 整体流程
```c
#include <windows.h>

// 函数提前声明 
LRESULT CALLBACK WndProc(HWND hwnd, UINT Message, WPARAM wParam, LPARAM lParam);

/* Win32界面应用的入口函数：程序将从此入口开始执行  */
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
	WNDCLASSEX wc; /* 用于声明窗口属性的窗口类,WNDCLASSEX是WNDCLASS的拓展 */
	HWND hwnd;     /* 窗口句柄，指向我们创建的窗口 */
	MSG msg;       /* 存放消息循环过程中产生的消息 */

	/* 将结构体置为空，我们只修改结构体中的一部分 */
	memset(&wc,0,sizeof(wc));
	/* 结构体的第一个字段,主要用于GetClassInfoEx能直接得到结构体大小 */
	wc.cbSize		 = sizeof(WNDCLASSEX); 
	/* 这个函数指针是核心,指定了消息循环过程中消息最终给哪个回调函数处理 */
	wc.lpfnWndProc	 = WndProc;
	wc.hInstance	 = hInstance; /* 应用实例句柄 */
	wc.hCursor		 = LoadCursor(NULL, IDC_ARROW); /* 窗口使用的鼠标样式 */
	
	/* 指定窗口的背景颜色 */
	wc.hbrBackground = (HBRUSH)(COLOR_WINDOW+1);
	wc.lpszClassName = "WindowClass";                   /* 窗口类的名字*/
	wc.hIcon		 = LoadIcon(NULL, IDI_APPLICATION); /* 窗口图标 */
	wc.hIconSm		 = LoadIcon(NULL, IDI_APPLICATION); /* 窗口小图标 */

	/* 注册窗口类:WNDCLASSEX用RegisterClassEx注册,WNDCLASS用RegisterClass注册 */ 
	if(!RegisterClassEx(&wc)) {
		// 注册失败,退出程序 
		MessageBox(NULL, "Window Registration Failed!","Error!",MB_ICONEXCLAMATION|MB_OK);
		return 0;
	}

	/* 使用刚注册的窗口类创建窗口 */
	hwnd = CreateWindowEx(WS_EX_CLIENTEDGE,"WindowClass","Caption",WS_VISIBLE|WS_OVERLAPPEDWINDOW,
		CW_USEDEFAULT, /* 窗口x坐标 */
		CW_USEDEFAULT, /* 窗口y坐标 */
		640, /* 窗口宽度 */
		480, /* 窗口高度 */
		NULL,NULL,hInstance,NULL);

	if(hwnd == NULL) {
		// 窗口句柄为空说明窗口创建失败,退出程序 
		MessageBox(NULL, "Window Creation Failed!","Error!",MB_ICONEXCLAMATION|MB_OK);
		return 0;
	}

	/*
		消息循环机制是整个应用的核心，消息循环过程中产生的所有消息都会发送给WndProc 
		GetMessage方法会发生阻塞直到获取到消息后才会返回, 
		所以这个循环并不会产生不合理的高CPU占用。 
	*/
	while(GetMessage(&msg, NULL, 0, 0) > 0) { /* 会一直循环下去，直到接收到WM_QUIT消息 */
		/* 对消息进行翻译(比如把按键码翻译成对应的字符) */
		/* 如果程序更复杂一点可能还需要调用TranslateAccelerator翻译菜单快捷键*/
		TranslateMessage(&msg); 
		/* 把消息分发给窗口类中定义的消息回调函数,也就是WndProc */
		DispatchMessage(&msg);
	}
	// 正常结束,把WM_QUIT的wParam参数中的exitCode返回给操作系统 
	return msg.wParam;
}


/**
 * 窗口的所有消息都会被分发到这个回调函数中 
 * hwnd窗口句柄，指向当前窗口
 * message 窗口消息
 * wParam,lParam 消息的附加参数,每种消息参数的意义都有所不同
 * LRESULT返回消息处理的结果 
 */
LRESULT CALLBACK WndProc(HWND hwnd, UINT Message, WPARAM wParam, LPARAM lParam) {
	switch(Message) {
		
		/* 销毁窗口消息 */
		case WM_DESTROY: {
			// 往消息队列中发一个WM_QUIT消息,退出程序 
			PostQuitMessage(0);
			break;
		}
		
		/* 其他所有的消息直接交给Windows系统默认的回调函数处理 */
		default:
			return DefWindowProc(hwnd, Message, wParam, lParam);
	}
	return 0;
}
```
> 看到上面这段代码，倍感亲切啊

# Windows程序的入口
我们知道标准C的入口函数是`main`，对于Windows程序来说程序的入口函数是`WinMain`。
> 不要被`main`函数限制了你的想象力，`main`只是入口函数的标记，程序运行时会根据`main`所指向的函数地址找到要从哪条指令开始执行。我们编译链接的时候可以自行指定这个标记。
>
> 在VC的链接器中有**/SUBSYSTEM:CONSOLE**和**/SUBSYSTEM:WINDOWS**两个选项
>
> | 应用程序类型                          | 入口函数(入口点)     | 嵌入执行体的启动函数 |
> | ------------------------------------- | -------------------- | -------------------- |
> | 处理ANSI字符和字符串的GUI应用 程序    | _tWinMain (WinMain)  | WinMainCRTStartup    |
> | 处理Unicode字符和字符串的GUI应 用程序 | _tWinMain (wWinMain) | wWinMainCRTStartup   |
> | 处理ANSI字符和字符串的CUI应用 程序    | _tmain (Main)        | mainCRTStartup       |
> | 处理Unicode字符和字符串的CUI应 用程序 | _tmain (Wmain)       | wmainCRTStartup      |
>
> 如果指定了**/SUBSYSTEM:WINDOWS**链接器开关，链接器就会寻找**WinMain**或**wWinMain**函数。如果没有找到这两个函数，链接器将返回一个“unresolved external symbol(未解析的外部符号)”错误;否则，它将根据具体情况分别选择**WinMainCRTStartup**或**wWinMainCRTStartup**函数。
>
> [这篇文章](http://mosir.org/html/y2012/the-custom-start-address-of-c-program.html)演示了gcc和vc两种编译器自定义入口函数。

[`WinMain`](https://msdn.microsoft.com/EN-US/library/windows/desktop/ms633559.aspx)函数的定义如下：

```c
// 返回值为int类型,如果程序在进入消息循环之前就结束了,应该返回0
// 如果函数进入消息循环后,在收到WM_QUIT消息后结束程序,应该返回消息中的wParam参数
int CALLBACK WinMain(
  // 当前应用的实例句柄
  _In_ HINSTANCE hInstance,
  // 前一个应用的实例句柄,大部分情况下为NULL
  _In_ HINSTANCE hPrevInstance,
  // 当使用命令行启动应用时,这个参数可以获取完整的命令
  _In_ LPSTR     lpCmdLine,
  // 控制窗口如何显示,这个参数一般会传给ShowWindow
  _In_ int       nCmdShow
);
```

# 消息循环机制

[消息循环(message loop)机制](https://en.wikipedia.org/wiki/Message_loop_in_Microsoft_Windows)是整个Windows应用程序的核心。

Windows界面程序是基于事件驱动的，在启动一个进程后，操作系统会为它维护一个单独的消息队列。操作系统会将窗口上的操作以消息的形式放入消息队列，比如鼠标在窗口上移动，窗口获取焦点后键盘敲击，点击窗口的按钮等。
程序员通过调用`GetMessage`函数从消息队列中获取消息，如果队列中没有消息，`GetMessage`函数将会阻塞。然后调用`DispatchMessage`将消息交给程序员定义的消息回调函数`WndProc`处理。

> 注意：整个过程都是在主线程处理的，`WndProc`也会在主线程中回调，所以`WndProc`中不要执行一些耗时的操作，比如网络请求、耗时的计算，否则主线程阻塞在这，处理不了其他的消息，将会导致窗口“假死”。

![Windows消息循环机制](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjz8mccij20fs0oa0so.jpg)

> 消息循环机制不仅存在Windows程序中，其实安卓、Web这些带UI界面的程序本质上都是基于消息循环机制。

[`GetMessage`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644936.aspx)函数是阻塞式的，如果消息队列中没有消息这个函数会发生阻塞，这时循环是“静止”的，这往往意味着低的CPU占用，对系统、对其它应用程序都是友好的。

另外还有一个[`PeekMessage`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644943.aspx)的函数，它可以检测消息队列中是否有消息，如果没有消息它会返回而不是发生阻塞，如果有消息可以通过最后一个参数传递**`PM_NOREMOVE`**或**`PM_REMOVE`**决定是否从消息队列中移除消息。

```C
HWND hwnd; 
BOOL fDone; 
MSG msg; 
 
fDone = FALSE; 
while (!fDone) 
{ 
    fDone = DoAnyting(); // 用户定义处理其他事情
 
    // 如果队列中有消息就取出消息并从队列中移除
 
    while (PeekMessage(&msg, hwnd,  0, 0, PM_REMOVE)) 
    { 
        switch(msg.message) 
        { 
            case WM_LBUTTONDOWN: 
            case WM_RBUTTONDOWN: 
            case WM_KEYDOWN: 
                // 
                // 处理消息
                break;
            case WM_QUIT:
                fDone = TRUE; 
                break;
        } 
    }
}
```

所以游戏编程里经常会出现这样的代码(DX红龙书上的代码片段)：

```c++
MSG msg;
::ZeroMessage(&msg, sizeof(msg));

static float lastTime = (float)timeGetTime();
while(msg.message != WM_QUIT)
{
    // 游戏对帧频要求较高，需要不间断的切换缓冲页面
	if(::PeekMessage(&msg, 0, 0, 0, PM_REMOVE))
    {
         ::TranslateMessage(&msg);
         ::DispatchMessage(&msg);
    }
    else
    {
         float currTime = (float)timeGetTime();
         float timeDelta = (currTime - lastTime) * 0.001f;
         
         display(timeDelta); // 调用显示函数
         
         lastTime = currTime;
    }
}
```

参考：
* 消息与消息队列：https://msdn.microsoft.com/EN-US/library/windows/desktop/ms632590.aspx

# 回调函数-消息处理函数

上面WinMain的代码，90%以上都是“样板代码”：通常创建一个Windows窗口应用就必须按照这个流程，可变化的地方很少。

> 创建一个窗口要写这么多“样板代码”，很明显上面的代码是需要封装的。[MFC](https://en.wikipedia.org/wiki/Microsoft_Foundation_Class_Library)，[Duilib](https://github.com/duilib/duilib)中都有对这些“样板代码”的封装。

我们实际要干的活都在WndProc这个回调函数中。

在让我们认识一下这个回调函数：

```c
LRESULT CALLBACK WndProc(HWND hwnd, UINT Message, WPARAM wParam, LPARAM lParam) {
		return DefWindowProc(hwnd, Message, wParam, lParam);
}
```

窗口回调函数有四个参数，事实上这四个参数都来[`MSG`这个消息结构体](https://msdn.microsoft.com/EN-US/library/windows/desktop/ms644958.aspx)，因为窗口回调函数就是用来处理消息的：

```c
typedef struct tagMSG {
  // 窗口句柄,窗口的处理函数会接受这个消息
  HWND   hwnd;
  // 消息ID
  UINT   message;
  // wParam和lParam都是消息附加信息，
  // 具体值取决于具体的消息(不同消息附加内容意义不一样)
  WPARAM wParam;
  LPARAM lParam;
  // 消息发出的时间
  DWORD  time;
  // 消息发出时光标的位置
  POINT  pt;
} MSG, *PMSG, *LPMSG;
```

message是消息的ID号，UNIT类型的有四个字节，不过能用的只有两个低字节(0x0000~0xFFFF)，高字节被系统保留。
**消息主要分为两种：系统消息和用户自定义消息**
* 系统消息：0x0000 ~ 0x03FF([`WM_USER`](https://msdn.microsoft.com/EN-US/library/windows/desktop/ms644931.aspx)-1)
* 用户自定义消息：用户自定义消息也分两种
  * 窗口类私有消息：0x0400(`WM_USER`) ~ 0x7FFF([`WM_APP`](https://msdn.microsoft.com/EN-US/library/windows/desktop/ms644930.aspx)-1)
  * 应用程序私有消息：0x8000(`WM_APP`) ~ 0xBFFF
> 还有一种消息范围在0xC000 ~ 0xFFFF，这个范围内的消息是通过[`RegisterWindowMessage`](https://msdn.microsoft.com/EN-US/library/windows/desktop/ms644947.aspx)函数进行注册得到的，这些消息ID能保证在整个操作系统是唯一的。

大多数情况下我们都是对系统消息进行处理，这些系统消息的ID都定义在在`WinUser.h`头文件中。

```c
/*
 * Window Messages
 */

#define WM_NULL                         0x0000
#define WM_CREATE                       0x0001
#define WM_DESTROY                      0x0002
#define WM_MOVE                         0x0003
#define WM_SIZE                         0x0005
...
```

消息很多，每个消息的附加参数意义也不一样，你不可能靠脑子去记的，所以需要经常查文档：https://msdn.microsoft.com/EN-US/library/windows/desktop/ms644927.aspx#system_defined

参考：
* 消息处理函数：https://msdn.microsoft.com/en-us/library/windows/desktop/ms632593.aspx

# 句柄与指针

前面频繁出现的一个东西HWND——窗口句柄。当然除了窗口句柄(HWND)，还有应用实例句柄(HINSTANCE)、文件句柄(HFILE)等各种句柄。**有了这些句柄我们就可以操作对应的Windows对象**。

前面为了方便理解，我们把它叫做指针。每一个程序员都有一颗好奇的心，你肯定尝试过使用这个“指针”去看看它指向的那段内存到底存的是不是窗口或者应用实例，但现实让你碰了一鼻子灰——句柄和指针还是不同的。

> 指针指向系统中物理内存的地址，而句柄是windows在内存中维护的一个对象内存物理地址列表的整数索引，句柄是一种指向指针的指针

使用Windows对象的句柄规范对系统资源的访问，这主要有两个原因：
* 给你一个Windows对象句柄而不是整个对象，是因为微软更新Windows系统时可能会修改Windows对象的内部结构(比如多加两个字段)，但是API中全部使用句柄，那么**系统维护的时候就可以尽可能少的修改API接口，让Windows开发者也尽可能少的修改代码或者不修改代码**。
* 为了系统安全性，Windows内部为每个对象维护了一个访问控制列表(ACL)，只有指定进程可以在对象上操作，每次为对象创建句柄的时候，系统都会去检查对象的ACL。

> 关于Windows ACL的详细内容可以参考：https://msdn.microsoft.com/en-us/library/windows/desktop/aa374860.aspx
> 对指针和句柄的理解，我个人觉的这篇文章讲的非常好，言简意赅：https://blog.csdn.net/u014041012/article/details/44878375

参考：

* [Windows核心编程](https://book.douban.com/subject/3235659/)

* 句柄与Windows对象：https://msdn.microsoft.com/en-us/library/windows/desktop/ms724457.aspx