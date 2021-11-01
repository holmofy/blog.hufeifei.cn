---
title: typedef与#define的区别
date: 2017-07-25
categories: C&C++
tags: 
- C
- Nginx
- VSCode
keywords:
- 调试
- Nginx
- VSCode
---

![NGINX architecture](http://www.aosabook.org/images/nginx/architecture.png)

vscode调试nginx源码

## clone源码

```bash
git clone https://github.com/nginx/nginx
```

除了官方的nginx，也可以考虑用阿里的[Tengine](https://github.com/alibaba/tengine)或[OpenResty](https://github.com/openresty/ngx_openresty)。

这两个发行版添加了各自的[三方module](https://github.com/agile6v/awesome-nginx#third-party-modules)。

## 编译运行

1. 修改 /auto/cc/conf 文件，将ngx_compile_opt="-c" 修改为 ngx_compile_opt="-c -g"

   > `-g`用来生成调试信息：详见[gcc文档](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html)

2. 执行 sudo ./auto/configure --prefix=nginx工程目录 ，如果遇到错误 "the HTTP rewrite module requires the PCRE library"，说明少了用来匹配正则表达式的`pcre`依赖包，可以自行根据平台进行安装

   ```debein
   apt install pcre2-utils
   ```

   ```macos
   brew install pcre
   ```

   ```centos
   yum install pcre
   ```

   

3. 执行 sudo make

4. 执行 ./objs/nginx，打开浏览器访问下 127.0.0.1，没问题的话就可以看到Nginx的欢迎界面了。

   ![nginx](https://s.pc.qq.com/tousu/img/20211101/7899594_1635743831.jpg)

> 具体源码编译内容可以参考[nginx文档](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/#compiling-and-installing-from-source)

## Nginx的多进程架构

nginx是多进程架构：一个Master进程，若干个Worker进程。

Master进程负责管理 Worker 进程，处理nginx命令行指令

Worker进程负责接收处理客户端请求

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/19/1705d5bae51f9935~tplv-t2oaga2asx-watermark.awebp)

Worker进程数通常设置成CPU核数，[`worker_processes: auto;`](https://nginx.org/en/docs/ngx_core_module.html#worker_processes)可以自动检查CPU设置成核心数。

Worker进程和redis类似使用单线程+IO多路复用实现高并发处理IO请求。

![NGINX architecture](http://www.aosabook.org/images/nginx/architecture.png)

## Master进程调试

1. 修改`/conf/nginx.conf`

   ```nginx
   # 关闭Master守护进程的功能
   daemon off;
   # 便于调试只启动一个Worker进程
   worker_processes  1;
   ```

   daemon默认都是`on`，[开发阶段关闭](http://nginx.org/en/docs/ngx_core_module.html#daemon)

2. 添加VSCODE调试配置

   ```JSON
   {
     "version": "0.2.0",
     "configurations": [
       {
         "name": "(gdb) Launch",
         "type": "cppdbg",
         "request": "launch",
         "program": "${workspaceFolder}/objs/nginx",
         "args": [
           "-c",
           "${workspaceFolder}/conf/nginx.conf"
         ],
         "stopAtEntry": false,
         "cwd": "${workspaceFolder}",
         "environment": [],
         "MIMode": "gdb",
         "miDebuggerPath": "/usr/bin/gdb",
         "setupCommands": [
           {
             "description": "Enable pretty-printing for gdb",
             "text": "-enable-pretty-printing",
             "ignoreFailures": true
           }
         ]
       }
     ]
   }
   ```

   如果是MacOS，不愿意装`gdb`，也可以用llvm的`lldb`进行调试。具体配置可以参考[vscode文档](https://code.visualstudio.com/docs/cpp/launch-json-reference#_customizing-gdb-or-lldb)

3. 打断点，Debug起来

   ![nginx debug](https://s.pc.qq.com/tousu/img/20211101/1323853_1635750161.jpg)

## 调试Worker进程

1. 查看 Worker 进程pid

   ```bash
   ps aux | grep nginx
   ```

   ![nginx worker process](https://s.pc.qq.com/tousu/img/20211101/5578379_1635750721.jpg)

2. 编辑`launch.json`，Attach到worker进程

   ```json
   {
     "version": "0.2.0",
     "configurations": [
   	/* ... */
       {
         "name": "(gdb) Attach Worker",
         "type": "cppdbg",
         "request": "attach",
         "program": "${workspaceFolder}/objs/nginx",
         "MIMode": "gdb",
         "miDebuggerPath": "/usr/bin/gdb",
         "processId": "9133"
       }
     ]
   }
   ```

   这里的进程id填进去就行了

3. 切换到`Attach Worker`

   ![Attach Worker](https://s.pc.qq.com/tousu/img/20211101/1899231_1635751242.jpg)

4. 接受请求的地方打个断点，浏览器重新刷新一下，请求就进来了

   ![Debug Worker Process](https://s.pc.qq.com/tousu/img/20211101/5407806_1635751394.jpg)

5. 从函数堆栈中，可以看到请求的处理过程

   ![process request](https://s.pc.qq.com/tousu/img/20211101/6907176_1635751804.jpg)

