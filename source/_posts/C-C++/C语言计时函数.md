---
title: C语言计时函数
date: 2017-07-28
categories: C&C++
tags: 
- C
---

# time()函数与time_t类型

**头文件**：time.h

**函数签名**：`time_t time( time_t *arg )`

**说明**：返回当前计算机纪元时间，并将其存储在arg指向的time_t类型中。所以可以`time_t result = time(NULL)`，也可以`time(&result)`。

> 计算机纪元时间：C语言和Unix创造并诞生于1970年，所以计算机以1970年1月1日作为纪元开始时间。

C语言标准并没有指定time_t类型的编码方式，但大多数遵循POSIX标准系统的time_t一般是32位有符号整数实现，以秒为最小单位，从1970年1月1日开始计数，所以能表示到2038年。

VS2017中有如下定义：

```c
#ifndef _CRT_NO_TIME_T
    #ifdef _USE_32BIT_TIME_T
        typedef __time32_t time_t;
    #else
        typedef __time64_t time_t;
    #endif
#endif

...
#ifdef _USE_32BIT_TIME_T
	...
	    static __inline time_t __CRTDECL time(
        _Out_opt_ time_t* const _Time
        )
    {
        return _time32(_Time);
    }
	...
#else
	...
    static __inline time_t __CRTDECL time(
        _Out_opt_ time_t* const _Time
        )
    {
        return _time64(_Time);
    }
	...
#endif
```

因为time_t类型编码不能确定，所以尽量不要用`t1-t2`方式计算两个`time_t`之间的时间间隔，而应该用`double difftime( time_t time_end, time_t time_beg );`函数计算时间间隔。

**扩展**：`time_t`表示计算机纪元时间，`struct tm`表示标准日历时间。

`struct tm`定义如下：

```c
struct tm
{
    int tm_sec;   // seconds after the minute - [0, 60] including leap second
    int tm_min;   // minutes after the hour - [0, 59]
    int tm_hour;  // hours since midnight - [0, 23]
    int tm_mday;  // day of the month - [1, 31]
    int tm_mon;   // months since January - [0, 11]
    int tm_year;  // years since 1900
    int tm_wday;  // days since Sunday - [0, 6]
    int tm_yday;  // days since January 1 - [0, 365]
    int tm_isdst; // daylight savings time flag
};
```

`time_t`可以和`struct tm`格式的时间进行互相转换：

* `struct tm *gmtime( const time_t *time )`：从`time_t`转成`struct tm`，但该函数C11标准中才定义的。
* `time_t mktime( struct tm *time )`：从`struct tm`转成`time_t`。

**示例**：

```c
	time_t start, end;
	start = time(NULL);
	_sleep(1000);
	end = time(NULL);
	printf("start time: %s\n", ctime(&start));
	printf("end time: %s\n", ctime(&end));
	printf("duration: %lf\n", difftime(end, start));
```

运行结果：

```shell
start time: Fri Jul 28 09:48:25 2017

end time: Fri Jul 28 09:48:26 2017

duration: 1.000000
```

**总结**：C标准库中的函数，可移植性最好，性能也很稳定，但精度太低，只能精确到秒，对于一般的事件计时还算够用，而对运算时间的计时就明显不够用了。

# clock()函数与clock_t类型

**头文件**：time.h

**函数签名**：`clock_t clock(void)`

**说明**：该函数返回值是硬件滴答数（未加工的程序启动时开始经过的处理器时间），只有计算调用两次clock的返回值才有意义，要换算成秒，需要除以CLOCKS_PER_SEC（每秒中的滴答数），VC中定义如下：

```c
#define CLOCKS_PER_SEC  ((clock_t)1000)
```

当CPU被多个进程共享，clock返回值可能慢于真实时钟，而如果一个进程内有多个线程或者多核处理，clock返回值可能快于真实时钟。

**注意**：在32位机器上clock_t为32bit，所以clock_t在`2147483648`(2147秒或36分)后会回滚(即小于零)。

**示例**：

示例1

```c
	clock_t start, end;
	start = clock();
	_sleep(1234);
	end = clock();
	double duration = ((double)end - start) / CLOCKS_PER_SEC;
	printf("%f second", duration);
```

运行结果：

```shell
1.234000 second
```

示例二：

```c
	clock_t start, end;
	start = clock();
	_sleep(1234);
	end = clock();
	int duration = end - start;
	printf("%d millisecond", duration);
```

运行结果：

```shell
1234 millisecond
```

**总结**：可以精确到毫秒，适合一般场合的使用。

# timespec_get()函数与timespec类型

**头文件**：time.h

**函数签名**：`int timespec_get( struct timespec *ts, int base)` （since C11）

**说明**：以base为基底，将当前计算机纪元时间填充到ts地址中。

timespec结构体定义如下：

```c
    struct timespec
    {
        time_t tv_sec;  // Seconds - >= 0
        long   tv_nsec; // Nanoseconds - [0, 999999999]
    };
```

timespec精度是纳秒级。

C11标准中定义`TIME_UTC`为通用协调时间，并推荐使用TIME_UTC作为base基底。

**示例**：

```c
	struct timespec start, end;
	timespec_get(&start, TIME_UTC);
	_sleep(1234);
	timespec_get(&end, TIME_UTC);

	char buffer[128];
	strftime(buffer, sizeof(buffer), "%Y/%m/%d %T", gmtime(&start.tv_sec));
	printf("start time: %s.%ld\n", buffer, start.tv_nsec);
	strftime(buffer, sizeof(buffer), "%Y/%m/%d %T", gmtime(&end.tv_sec));
	printf("end time: %s.%ld\n", buffer, end.tv_nsec);

	time_t d_sec = end.tv_sec - start.tv_sec;
	long d_nsec = end.tv_nsec - start.tv_nsec;
	long duration = d_sec*1E9 + d_nsec;
	printf("duration: %ld nanosecond\n", duration);
```

运行结果：

```shell
start time: 2017/07/28 04:34:26.577550200
end time: 2017/07/28 04:34:27.811685800
duration: 1234135600 nanosecond
```

**总结**：C11标准库函数，跨平台特性好，精度能达到纳秒级，但是需要编译器支持C11。



---

以上是C语言标准库提供的函数，跨平台特性较好，下面的几个是特定平台的API，所以视情况使用。

# timeGetTime()函数

**头文件**：timeapi.h（Windows.h中已经包括该头文件）

**函数签名**：`DWORD timeGetTime(void)`

**库**：Winmm.lib；**Dll**：Winmm.dll

**说明**：返回系统时间，以毫秒为单位。

timeGetTime函数返回值是一个DWORD(32bit)，会在0到$2^{32}$之间循环，大约49.71天。如果使用这个返回值作为控制语句的条件可能会出现问题，所以建议使用这个函数计算两个时间点之间的间隔。

timeGetTime函数的默认精度可以达到5毫秒或者更多，这个精度依赖于机器。可以使用timeBeginPeriod和timeEndPeriod函数增加timeGetTime的精度。timeGetTime返回的两个连续值之间的最小间隔可以和timeBeginPeriod和timeEndPeriod设置的最小周期一样大。

> [**timeGetSystemTime**](https://msdn.microsoft.com/zh-cn/library/windows/desktop/dd757628(v=vs.85).aspx)与timeGetTime的区别：
>
> timeGetTime以DWORD存储时间
>
> timeGetSystemTime以[**MMTIME**](https://msdn.microsoft.com/zh-cn/library/windows/desktop/dd757347(v=vs.85).aspx)结构体存储时间

**示例**：

示例一：

```c
#include<Windows.h>
#pragma comment(lib,"Winmm.lib")
int main() {
	DWORD start, end;
	start = timeGetTime();
	Sleep(1234);
	end = timeGetTime();
	printf("%d\n", end - start);
}
```

运行结果：

```shell
1235
```

示例二：

```c
#include<Windows.h>
#pragma comment(lib,"Winmm.lib")
int main() {
	DWORD start, end;
	timeBeginPeriod(1);
	start = timeGetTime();
	Sleep(1234);
	end = timeGetTime();
	timeEndPeriod(1);
	printf("%d\n", end - start);
}
```

运行结果：

```shell
1234
```

# GetTickCount函数

**头文件**：Winbase.h（Windows.h中已经包括该头文件）

**函数签名**：DWORD WINAPI GetTickCount(void);

**库**：Kernel32.lib；**Dll**：Kernel32.dll

**说明**：返回系统启动后的时间，单位为毫秒。

GetTickCount功能只限于系统定时器，精度在10毫秒到16毫秒之间。GetTickCount的精度不受GetSystemTimeAdjustment函数影响。

该函数以DWORD(32bit)存储时间，如果系统连续运行49.7天，将会循环到0，为避免这个影响，建议使用GetTickCount64函数

**示例**：

```c
#include<Windows.h>

int main() {
	DWORD start, end;
	start = GetTickCount();
	Sleep(1234);
	end = GetTickCount();
	printf("%d\n", end - start);
}
```

运行结果：

```shell
1234
```

# QueryPerformanceCounter()、QueryPerformanceFrequency()高精度时间

**头文件**：Winbase.h (Windows.h中已经包括该头文件)

**函数签名**：

* ```
  BOOL WINAPI QueryPerformanceCounter(
    _Out_ LARGE_INTEGER *lpPerformanceCount
  );
  ```

* ```
  BOOL WINAPI QueryPerformanceFrequency(
    _Out_ LARGE_INTEGER *lpFrequency
  );
  ```

**库**：Kernel32.lib；**Dll**：Kernel32.dll

**说明**：QueryPerformanceFrequency获取CPU时钟频率；QueryPerformanceCounter获取CPU时钟计数器的当前值。通过这两个函数可以计算两点之间的时间间隔，精度为微妙级别。

LARGE_INTEGER定义如下：

```c
typedef union _LARGE_INTEGER {
    struct {
        DWORD LowPart;
        LONG HighPart;
    } DUMMYSTRUCTNAME;
    struct {
        DWORD LowPart;
        LONG HighPart;
    } u;
    LONGLONG QuadPart;
} LARGE_INTEGER;
```

**示例**：

```c
#include<Windows.h>

DWORD64 ObtainCurrentTime() {
	LARGE_INTEGER num;
	QueryPerformanceCounter(&num);
	return num.QuadPart;
}

DOUBLE ComputeDuration(DWORD64 end, DWORD64 start) {
	LARGE_INTEGER frequency;
	QueryPerformanceFrequency(&frequency);
	return (DOUBLE)(end - start) / frequency.QuadPart;
}

int main() {
	DWORD64 start, end;
	start = ObtainCurrentTime();
	Sleep(1234);
	end = ObtainCurrentTime();
	DOUBLE duration = ComputeDuration(end, start);
	printf("%lf microsecond", duration);
}
```

运行结果：

```shell
1.234634 microsecond
```



> 更多方法可以参考http://blog.csdn.net/fz_ywj/article/details/8109368

> **参考链接：**
>
> * C Reference：http://en.cppreference.com/w/c/chrono
> * MSDN-多媒体定时器：https://msdn.microsoft.com/en-us/library/windows/desktop/dd743612(v=vs.85).aspx
> * MSDN-高精度定时器：https://msdn.microsoft.com/en-us/library/windows/desktop/ms644900(v=vs.85).aspx
> * MSDN-Windows时间：https://msdn.microsoft.com/en-us/library/windows/desktop/ms725496(v=vs.85).aspx
> * MSDN-时间相关函数：https://msdn.microsoft.com/en-us/library/windows/desktop/ms725473(v=vs.85).aspx
> * CSDN-常用计时方法总结：http://blog.csdn.net/fz_ywj/article/details/8109368

