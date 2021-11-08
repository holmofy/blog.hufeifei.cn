---
title: 在nginx中安装并调试OpenResty
date: 2021-11-01
categories: C&C++
tags: 
- C
- Nginx
- VSCode
- OpenResty
keywords:
- 调试
- Nginx
- VSCode
- OpenResty
---

### 安装OpenResty相关模块

OpenResty是基于Lua即时编译器([LuaJIT](https://github.com/LuaJIT/LuaJIT))对Nginx进行扩展的模块——最核心的就是[`lua-nginx-module`](https://github.com/openresty/lua-nginx-module)这个模块。其他的都是[OpenResty基于lua开发的相关模块](https://github.com/bungle/awesome-resty)，当然也可以基于lua开发自己的第三方模块。

所以要想使用OpenResty首先必须安装`lua-nginx-module`。

1. 下载并安装LuaJIT。可以使用源码方式安装，这个可以参考[官方文档](https://luajit.org/install.html)非常详细。这里为了方便直接用apt安装了

   ```bash
   sudo apt install luajit libluajit-5.1-dev
   ```

2. 下载`ngx_devel_kit`模块

   ```bash
   # 在nginx目录下创建一个modules目录
   mkdir modules
   # 从github克隆模块代码
   git clone https://github.com/vision5/ngx_devel_kit/ modules/ngx_devel_kit
   # 切换到v0.3.1版本
   git checkout tags/v0.3.1 -b v0.3.1
   ```

3. 下载`lua-nginx-module`模块

   ```bash
   git clone https://github.com/openresty/lua-nginx-module modules/ngx_http_lua_module
   git checkout tags/v0.10.20 -b v0.10.20
   ```

   如果是使用alibaba/tengine，这个模块已经被包含在tengine的`modules/ngx_http_lua_module`目录下了。

   > 另外注意[lua-nginx-module与nginx的兼容性](https://github.com/openresty/lua-nginx-module#nginx-compatibility)，nginx1.6.0之前的版本是不支持的。

### 编译nginx源码

0. 如果需要对nginx进行debug的话，需要修改 /auto/cc/conf 文件，将`ngx_compile_opt="-c"`修改为 `ngx_compile_opt="-c -g"`

   > `-g`用来生成调试信息：详见[gcc文档](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html)

1. 设置luajit的头文件和静态库的路径，ubuntu下可以用dpkg看看libluajit被安装到哪个目录了

   ```bash
   $ dpkg -L libluajit-5.1-dev
   /.
   /usr
   /usr/include
   /usr/include/luajit-2.1
   /usr/include/luajit-2.1/lauxlib.h
   /usr/include/luajit-2.1/lua.h
   /usr/include/luajit-2.1/lua.hpp
   /usr/include/luajit-2.1/luaconf.h
   /usr/include/luajit-2.1/luajit.h
   /usr/include/luajit-2.1/lualib.h
   /usr/lib
   /usr/lib/x86_64-linux-gnu
   /usr/lib/x86_64-linux-gnu/libluajit-5.1.a
   /usr/lib/x86_64-linux-gnu/pkgconfig
   /usr/lib/x86_64-linux-gnu/pkgconfig/luajit.pc
   /usr/share
   /usr/share/doc
   /usr/share/doc/libluajit-5.1-dev
   /usr/share/doc/libluajit-5.1-dev/copyright
   /usr/lib/x86_64-linux-gnu/libluajit-5.1.so
   /usr/share/doc/libluajit-5.1-dev/changelog.Debian.gz
   ```

   然后设置两个环境变量

   ```bash
   export LUAJIT_LIB=/usr/lib/x86_64-linux-gnu/
   export LUAJIT_INC=/usr/include/luajit-2.1/
   ```

2. 执行`auto/configure`

   ```bash
   auto/configure --prefix=nginx \
            --with-ld-opt="-Wl,-rpath,/usr/lib/x86_64-linux-gnu/" \
            --add-module=./modules/ngx_devel_kit \
            --add-module=./modules/ngx_http_lua_module
   ```

3. 执行`make install`

   > 编译过程可能会比较慢，可以执行`make -j2 && make install`调大编译任务的个数

### 调试OpenResty中的lua代码

这里以一个第三方的lua模板引擎为例——[lua-resty-template](https://github.com/bungle/lua-resty-template)

#### 安装lua模块

```bash
# 在nginx下创建一个放lua脚本的目录
mkdir lua-lib
# 下载lua-resty-template模块
git clone https://github.com/bungle/lua-resty-template lua-lib/lua-resty-template
```

在`nginx.conf`中对lua模块进行配置

```nginx
http {
    # ...
    # 设置lua模块的路径
    lua_package_path "lua-lib/lua-resty-template/lib/?.lua;;";
    
    server {
        listen       80;
        server_name  localhost;
        
        set $template_root html/templates;
        location /templates/ {
          root html;
          content_by_lua '
            local template = require "resty.template"
            template.render("view.html", { message = "Hello, World!" })
          ';      
    	}
    }
}
```

在`html/templates`目录下添加模板文件

```html
<!DOCTYPE html>
<html>
<body>
  <h1>{{message}}</h1>
</body>
</html>
```

访问`localhost/templates/view.html`，能看到下面的结果

![lua template](https://s.pc.qq.com/tousu/img/20211101/1341156_1635761461.jpg)

> Read More:
>
> https://github.com/lua/lua
>
> https://www.lua.org/pil/23.html
>
> http://lua-users.org/wiki/DebuggingLuaCode
>
> http://notebook.kulchenko.com/zerobrane/debugging-openresty-nginx-lua-scripts-with-zerobrane-studio
>
> https://github.com/LewisJEllis/awesome-lua

