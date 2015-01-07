# HttpLuaModule
==

## Name
ngx_lua - Embed the power of Lua into Nginx
这个扩展不随nginx的源码发布。[See the installation instructions.](http://wiki.nginx.org/HttpLuaModule#Installation)

## Status
可以用于生产环境

## Version
This document describes [ngx_lua v0.9.12](https://github.com/openresty/lua-nginx-module/tags) released on 2 September 2014.

## Synopsis

```
# set search paths for pure Lua external libraries (';;' is the default path):
lua_package_path '/foo/bar/?.lua;/blah/?.lua;;';
 
# set search paths for Lua external libraries written in C (can also use ';;'):
lua_package_cpath '/bar/baz/?.so;/blah/blah/?.so;;';
 
server {
    location /inline_concat {
        # MIME type determined by default_type:
        default_type 'text/plain';
        set $a "hello";
        set $b "world";
        # inline Lua script
        set_by_lua $res "return ngx.arg[1]..ngx.arg[2]" $a $b;
        echo $res;
    }
 
    location /rel_file_concat {
        set $a "foo";
        set $b "bar";
        # script path relative to nginx prefix
        # $ngx_prefix/conf/concat.lua contents:
        #
        #    return ngx.arg[1]..ngx.arg[2]
        #
        set_by_lua_file $res conf/concat.lua $a $b;
        echo $res;
    }
    location /abs_file_concat {
        set $a "fee";
        set $b "baz";
        # absolute script path not modified
        set_by_lua_file $res /usr/nginx/conf/concat.lua $a $b;
        echo $res;
    }
 
    location /lua_content {
        # MIME type determined by default_type:
        default_type 'text/plain';
        content_by_lua "ngx.say('Hello,world!')";
    }
 
     location /nginx_var {
        # MIME type determined by default_type:
        default_type 'text/plain';
 
        # try access /nginx_var?a=hello,world
        content_by_lua "ngx.print(ngx.var['arg_a'], '\\n')";
    }
 
    location /request_body {
        # force reading request body (default off)
        lua_need_request_body on;
        client_max_body_size 50k;
        client_body_buffer_size 50k;
 
        content_by_lua 'ngx.print(ngx.var.request_body)';
    }
 
    # transparent non-blocking I/O in Lua via subrequests
    location /lua {
        # MIME type determined by default_type:
        default_type 'text/plain';
 
        content_by_lua '
            local res = ngx.location.capture("/some_other_location")
            if res.status == 200 then
                ngx.print(res.body)
            end';
    }
 
    # GET /recur?num=5
    location /recur {
        # MIME type determined by default_type:
        default_type 'text/plain';
        content_by_lua '
            local num = tonumber(ngx.var.arg_num) or 0
            if num > 50 then
                ngx.say("num too big")
                return
            end
 
            ngx.say("num is: ", num)
 
            if num > 0 then
                res = ngx.location.capture("/recur?num=" .. tostring(num - 1))
                ngx.print("status=", res.status, " ")
                ngx.print("body=", res.body)
            else
                ngx.say("end")
            end
        ';
    }
 
    location /foo {
        rewrite_by_lua '
            res = ngx.location.capture("/memc",
                { args = { cmd = "incr", key = ngx.var.uri } }
            )
        ';
 
        proxy_pass http://blah.blah.com;
    }
 
    location /blah {
        access_by_lua '
            local res = ngx.location.capture("/auth")
 
            if res.status == ngx.HTTP_OK then
                return
            end
 
            if res.status == ngx.HTTP_FORBIDDEN then
                ngx.exit(res.status)
            end
 
            ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        ';
 
        # proxy_pass/fastcgi_pass/postgres_pass/...
    }
 
    location /mixed {
        rewrite_by_lua_file /path/to/rewrite.lua;
        access_by_lua_file /path/to/access.lua;
        content_by_lua_file /path/to/content.lua;
    }
 
    # use nginx var in code path
    # WARN: contents in nginx var must be carefully filtered,
    # otherwise there'll be great security risk!
    location ~ ^/app/(.+) {
        content_by_lua_file /path/to/lua/app/root/$1.lua;
    }
 
    location / {
        lua_need_request_body on;
 
        client_max_body_size 100k;
        client_body_buffer_size 100k;
 
        access_by_lua '
            -- check the client IP address is in our black list
            if ngx.var.remote_addr == "132.5.72.3" then
                ngx.exit(ngx.HTTP_FORBIDDEN)
            end
 
            -- check if the request body contains bad words
            if ngx.var.request_body and
                string.match(ngx.var.request_body, "fsck")
            then
                return ngx.redirect("/terms_of_use.html")
            end
            -- tests passed
        ';
        # proxy_pass/fastcgi_pass/etc settings
    }
}
```

## Description

本模块嵌入lua,将标准的Lua5.1的解释器或者LuaJIT2.0/2.1纳入Nginx，并且利用Nginx的子请求，将强大的Lua线程（Lua携程）融入Nginx的事件模型中

不像Apache的[mod_lua](https://httpd.apache.org/docs/trunk/mod/mod_lua.html)以及Lighttpd的[mod_magnet](http://redmine.lighttpd.net/wiki/1/Docs:ModMagnet)，在这个模块中，Lua代码的执行在网络传输上可以100%的无阻塞，只要使用此模块提供的[lua的nginx api](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)就可以处理一些上游服务的请求，例如：MySQL,PostgreSQL,Memcached,Redis，或者上游HTTP web services。

至少下面罗列的Lua库和Nginx模块可以使用此ngx_lua模块

* [**lua-resty-memcached**](https://github.com/openresty/lua-resty-memcached)
* [**lua-resty-mysql**](https://github.com/openresty/lua-resty-mysql)
* [**lua-resty-redis**](https://github.com/openresty/lua-resty-redis)
* [**lua-resty-dns**](https://github.com/openresty/lua-resty-dns)
* [
* 
* *ua-resty-upload**](https://github.com/openresty/lua-resty-upload)
* [**lua-resty-websocket**](https://github.com/openresty/lua-resty-websocket)
* [**lua-resty-lock**](https://github.com/openresty/lua-resty-lock)
* [**lua-resty-string**](https://github.com/openresty/lua-resty-string)
* [**ngx_memc**](http://wiki.nginx.org/HttpMemcModule)
* [**ngx_postgres**](https://github.com/FRiCKLE/ngx_postgres)
* [**ngx_redis2**](http://wiki.nginx.org/HttpRedis2Module)
* [**ngx_redis**](http://wiki.nginx.org/HttpRedisModule)
* [**ngx_proxy**](http://wiki.nginx.org/HttpRedisModule)
* [**ngx_fastcgi**](http://wiki.nginx.org/HttpFastcgiModule)

几乎所有的Nginx模块都可以通过[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture)或者[ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi)和ngx_lua模块一起使用，但是推荐使用lua-resty-*库，而不是创建子请求来访问Nginx的上游模块，因为前者通常更加灵活并且内存方面更加高效。

在一个单独的nginx的worker的所有请求中，Lua解释器或者LuaJIT实例是共享的，但是请求的上下文是分开使用轻量级的Lua协程。

加载的Lua模块常驻于nginx worker进程，这就造就了在重负荷情况下，对内存的需求量依然很小。

## Typical Uses

举一些例子：

*  使用Lua糅合处理nginx上游各种各样的输出（proxy，drizzle，postgres，redis，memcached，等等）
*  在请求到达后端之前，使用Lua做任意复杂的访问控制和安全检查
*  通过Lua任意操作响应头
*  通过从外部存储获取后端服务的配置信息等来决定从哪个上游的后端服务来获取数据
*  在任意复杂的web应用程序中使用同步，但对后端的数据库和其他存储的访问是无阻塞的
*  在rewrite阶段做非常复杂的URL的分发
*  使用Lua来为Nginx的子请求和任意的location实现高级缓存机制

这个模块汇集了Nginx内部的各种元素然后暴漏给用户，使得Lua的拥有无限的可能力量。nginx-lua模块提供了充分灵活的脚本的同时也提供了与C程序同等级的性能，无论是在CPU的时间上还是在内存的消耗上都同C的性能相当。尤其在LuaJIT 2.X打开了的时候。

其他的脚本语言的执行往往难以达到这种性能水平。

Lua是通过在在单独的nginx的worker进程中的所有请求共享同一Lua虚拟机实例来减少内存的开销。

## Nginx Compatibility

最新的模块与以下版本的Nginx兼容：

* 1.7.x (last tested: 1.7.4)
* 1.6.x
* 1.5.x (last tested: 1.5.12)
* 1.4.x (last tested: 1.4.4)
* 1.3.x (last tested: 1.3.11)
* 1.2.x (last tested: 1.2.9)
* 1.1.x (last tested: 1.1.5)
* 1.0.x (last tested: 1.0.15)
* 0.9.x (last tested: 0.9.4)
* 0.8.x >= 0.8.54 (last tested: 0.8.54)

## Installation

[ngx_openresty](http://openresty.org/)可以用来安装Nginx，ngx_lua，标准Lua5.1解释器或者LuaJIT 2.0/2.1，以及一些其他的强大的nginx模块。基本的安装步骤是一条简单的命令

```
./configure --with-luajit && make && make install
```

另外，ngx_lua也可以手工编译进nginx：

1. 安装LuaJIT2.0或者2.1（推荐）或者Lua5.1（Lua5.2现在不再支持）。LuaJIT可以从[LuaJIT官网](http://luajit.org/download.html)下载，Lua5.1从其[官网](http://www.lua.org/)可以下载
2. 下载最新版本的[ngx_devel_kit(NDK)](http://github.com/simpl/ngx_devel_kit/tags)模块
3. 下载最新的[ngx_lua](http://github.com/openresty/lua-nginx-module/tags)
4. 下载最新的[Nginx](http://nginx.org/)(注意上面提到的Nginx版本的兼容性)

一起编译安装：

```
    wget 'http://nginx.org/download/nginx-1.7.4.tar.gz'
    tar -xzvf nginx-1.7.4.tar.gz
    cd nginx-1.7.4/
 
    # tell nginx's build system where to find LuaJIT 2.0:
    export LUAJIT_LIB=/path/to/luajit/lib
    export LUAJIT_INC=/path/to/luajit/include/luajit-2.0
 
    # tell nginx's build system where to find LuaJIT 2.1:
    export LUAJIT_LIB=/path/to/luajit/lib
    export LUAJIT_INC=/path/to/luajit/include/luajit-2.1
 
    # or tell where to find Lua if using Lua instead:
    #export LUA_LIB=/path/to/lua/lib
    #export LUA_INC=/path/to/lua/include
 
    # Here we assume Nginx is to be installed under /opt/nginx/.
    ./configure --prefix=/opt/nginx \
            --add-module=/path/to/ngx_devel_kit \
            --add-module=/path/to/lua-nginx-module
 
    make -j2
    make install
```

### C Macro Configurations

无论是通过OpenResty还是独立的Nginx来安装这个模块，你都可以通过C编译器来定义下面的宏

* **NGX_LUA_USE_ASSERT** - 当被定义的时候，会打开C代码基础库中的断言。推荐调试或者测试用途的时候打开。当开启的时候它可以展示一些（小的）运行时的开销。这个宏在v0.9.10第一次被引入。
* **NGX_LUA_ABORT_AT_PANIC** - 当Lua/LuaJIT panics的时候，ngx_lua将默认的指示当前的nginx工作进程优雅的退出。通过指定此宏，ngx_lua在发生panics的时候立即中止当前的nginx工作进程（通常将结果存到一个core dump文件中）。这个选项对于调试VM panics相当有用。这个宏在v0.9.8第一次被引入。
* **NGX_LUA_NO_FFI_API** - 为Nginx的FFI-based Lua API排除纯C的API函数（例如，lua-resty-core的需求）。启用此宏可以使生成的二进制代码大小变小。

开启这些红中的一个或者多个，只需要传递额外的C编译选项给`./configure`脚本。例如：

```
./configure --with-cc-opt="-DNGX_LUA_USE_ASSERT -DNGX_LUA_ABORT_AT_PANIC"
```

### Installation on Ubuntu 11.10

推荐尽量使用LuaJIT 2.0或者2.1来代替标准的Lua5.1解释器。

如果一定要要使用标准的Lua 5.1解释器，使用下面的命令从Ubuntu的源来安装

```
apt-get install -y lua5.1 liblua5.1-0 liblua5.1-0-dev
```

一切都应安装正确，除了一个小的调整.
`liblua.so`库在liblua5.1包里面被改变了，它只提供了`liblua5.1.so`，所以需要将它连接到`/usr/lib`以便于在编译期间能被找到。

```
ln -s /usr/lib/x86_64-linux-gnu/liblua5.1.so /usr/lib/liblua.so
```

## Community

### English Mailing List
The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

### Chinese Mailing List
The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

## Code Repository

这个项目的代码库托管在github上面 [openresty/lua-nginx-module](http://github.com/openresty/lua-nginx-module)

## Bugs and Patches

请通过下面两种方式来提交bug报告，期望列表或者patches

1.在[Github Issue Tracker](https://github.com/openresty/lua-nginx-module/issues)上创建ticket
2.发送到OpenResty邮件组

## Lua/LuaJIT bytecode support

从v0.5.0rc32开始，所有的*_by_lua_file配置指令(例如content_by_lua_file)支持直接加载Lua5.1和LuaJIT2.0/2.1的原生字节码文件。

请注意，LuaJIT2.0/2.1与标准的Lua5.1解释器的字节码格式互不兼容。所以，如果使用的是LuaJIT2.0/2.1，必须使用下面的命令来生成LuaJIT的字节码文件

```
/path/to/luajit/bin/luajit -b /path/to/input_file.lua /path/to/output_file.luac
```

可以使用`-bg`选项来使字节码文件包含调试信息

```
/path/to/luajit/bin/luajit -bg /path/to/input_file.lua /path/to/output_file.luac
```

请到LuaJIT的官网查阅更多有关-b选项的详细信息 [http://luajit.org/running.html#opt_b](http://luajit.org/running.html#opt_b)

LuaJIT 2.1的字节码文件也不兼容LuaJIT 2.0的，反之亦然。ngx_lua在v0.9.3版第一次加入LuaJIT 2.1字节码的支持。

同样，如果使用标准的Lua 5.1解释器，Lua兼容的字节码文件必须使用下面的命令来生成

```
luac -o /path/to/output_file.luac /path/to/input_file.lua
```

不像LuaJIT，调试信息默认包含于标准Lua 5.1的字节码文件中。可以指定-s选项，按条罗列出来

```
luac -s -o /path/to/output_file.luac /path/to/input_file.lua
```

试图将标准Lua 5.1的字节码文件加载到LuaJIT 2.0/2.1运行的ngx_lua实例中，反之亦然，会产生一个错误信息，看下面的例子，被记录到`error.log`文件

```
[error] 13909#0: *1 failed to load Lua inlined code: bad byte-code header in /path/to/test_file.luac
```

通过Lua语法(require以及dofile)加载的字节码文件总是会按照预期的方式工作。

## System Environment Variable Support

如果你想在Lua中通过标准的API os.getenv来访问系统环境变量（例如say,foo等），那么你就需要在nginx.conf文件中通过env指令来列出系统变量的名字。例如

```
env foo;
```

## HTTP 1.0 support

HTTP 1.0协议不支持chunked分块输出，当响应体不为空的时候，引入了一个显示的头`Content-Length`来支持HTTP 1.0的keep-alive。所以当HTTP 1.0请求来到并且 [lua_http10_buffering](http://wiki.nginx.org/HttpLuaModule#lua_http10_buffering) 指令处于打开状态，ngx_lua会缓存由ngx.say和ngx.print的输出，等到所有的需要输出的响应体准备好之后才输出给用户。这种情况下，ngx_lua就能统计出响应体的总长度，并且构造一个合适的`Content-Length`头输出给HTTP 1.0的请求端。如果在运行的Lua代码中设置了`Content-Length`，那么，这个缓存就会被关闭，哪怕开启了`lua_http10_buffering`指令。

对于量比较大的响应，关闭`lua_http10_buffering`指令对于减小内存的消耗是非常重要的。

注意，通常的HTTP基准测试工具（ab、http_load等）默认都是HTTP 1.0协议的请求。若要强制`curl`发送HTTP 1.0的请求，使用`-o`选项。

## Statically Linking Pure Lua Modules

当使用LuaJIT 2.x的时候，可以将lua模块的字节码静态的链接到nginx的可执行文件。

基本上你可以使用luajit将.lua文件编程成包含字节码编译成.o对象文件，然后直接将.o文件链接到Nginx。

下面是一个简单的例子来做演示。我们将我们的.lua文件命名为foo.lua。

```
-- foo.lua
local _M = {}
 
function _M.go()
	print("Hello from foo")
end
 
return _M
```

接下来我们将这个.lua文件编译成foo.o文件：

```
/path/to/luajit/bin/luajit -bg foo.lua foo.o
```

这里最重要的是是.lua文件的名字，它决定了稍后你在在Lua代码中怎么使用本模块。foo.o这个文件的名字根本不重要，重要的是它是一个后缀为.o的文件（它将告诉luajit以何种格式输出）。如果你想过滤掉最终的字节码中的Lua调试信息，你只需要指定`-b`选项就行了。

接下来编译Nginx或者Openresty的时候，传递`--with-ld-option="foo.o"`
给`./configure`脚本：

```
./configure --with-ld-opt="/path/to/foo.o" ...
```

最后，你只需要按照下面的方式就可以在ngx_lua扩展中用Lua脚本来调用了

```
local foo = require "foo"
foo.go()
```

这小段代码不再依赖外部的`foo.lua`文件，因为它已经被编译到Nginx的可执行文件中。

如果你想在`require`调用的时候在Lua模块的名字中使用点，就像

```
local foo = require "resty.foo"
```

那么你需要在将它编译成一个.o文件之前将它的名字从`foo.lua`改成`resty_foo.lua`。

有一个非常重要的是将.lua文件编程成.o文件时使用的LuaJIT的版本要和你编译到Nginx中的nfx_lua的调用的LuaJIT的版本相同。这是因为在不同版本的LuaJIT字节码可能不兼容。当字节码格式不兼容的时候，你会看到一个Lua运行时的错误提示无法找到你的Lua模块。

如果你有多个.lua文件需要编译和链接，你只需要将它们的.o文件挨个指定到`--with-ld-opt`选项，例如：

```
./configure --with-ld-opt="/path/to/foo.o /path/to/bar.o" ...
```

如果你又很多的.o文件，将他们的名字都写到一条命令中将非常麻烦。这种情况下，你可以将你的.o文件生成一个静态链接库（或者存档），就像：

```
ar rcus libmyluafiles.a *.o
```

然后你就可以将`myluafiles`存档作为一个整体链接到nginx的可执行文件中：

```
./configure --with-ld-opt="-L/path/to/lib -Wl,--whole-archive -lmyluafiles -Wl,--no-whole-archive"
```

`/path/to/lib`是存放`libmyluafiles.a`文件的文件夹的路径。需要指出的是，连接器选项`--whole-archive`在这里是必须的，否则我们的存档会被忽略，因为在nginx的可执行文件的主要地方没有我们的存档的链接。

## Data Sharing within an Nginx Worker

在一个nginx的工作进程的所有请求中全局共享数据，将共享的数据封装到一个Lua模块，使用Lua内置的`require`来导入模块，然后在Lua中操作共享的数据。这么做的原因是Lua模块只加载一次，所有的协程将共享该模块完全相同的副本（包括它的代码和数据）。但是请注意，两个请求之间的Lua全局变量（注意：非模块级变量）将不会长久存在，这个原因是由于每个请求一个协程的分离设计。

下面是一个完整的小例子：

```
-- mydata.lua
local _M = {}
 
local data = {
	dog = 3,
	cat = 4,
	pig = 5,
}
 
function _M.get_age(name)
	return data[name]
end
 
return _M
```

从`nginx.conf`中访问他：

```
location /lua {
	content_by_lua '
		local mydata = require "mydata"
		ngx.say(mydata.get_age("dog"))
	';
}
```

这个例子中的`mydata`模块只会在`location /lua`第一次被请求时候被加载和运行，当前nginx工作进行中所有后续的请求将重新加载这个模块的实例以及相同的数据的副本，直到Nginx主进程收到一个`HUP`信号强迫它进行重载。这个数据共享技术是基于这个模块的Lua程序高性能的重要原因。

请注意，这个数据共享是在每个工作进程，而不是每个服务器。当一个Nginx的主进程下面有多个工作进程的时候，在这些工作进程之间不能进行数据的越界共访问。

通常建议使用这种方式来共享只读数据。当然你也可以在所有的nginx工作进程的并发请求之间共享可变数据，只要在你的代码进行的计算中没有非阻塞I/O操作（包括ngx.sleep）。只要你不把控制权还给nginx的事件循环和ngx_lua的轻量级线程调度（即使隐式的），在这两者之间决不能有任何的竞争条件。为此，当你在工作进程级别进行可变数据的共享的时候一定要非常小心。在高负荷情况下，古怪的代码优化可能会导致很难调试的竞争条件。

如果需要使用服务器级别的数据共享，请使用下面介绍的方法：

1.请使用ngx_lua模块提供的**ngx.shared.DICT** API
2.使用只有单一的工作进程和单独的服务器（不过不建议在多核CPU或者多个CPU的机器上使用）
3.使用一些第三方的存储机制，例如`memached`、`redis`、`MySQL`或者`PostgreSQL`。[`ngx_openresty`](http://openresty.org/)提供了这些存储的接口的库的封装。

## Known Issues

### TCP socket connect operation issues
[tcpsock:connect](http://wiki.nginx.org/HttpLuaModule#tcpsock:connect)方法在连接失败（`Connection Refused`）的情况下也会返回`success`。

然后，在接下来使用上面生成的cosocket对象的进行实际操作的时候将会失败，并返回实际的错误状态和信息。

此问题是由于Nginx的时间模型中的限制产生的，只会出现在Max OS X系统。

### Lua Coroutine Yielding/Resuming
* 在Lua 5.1以及LuaJIT 2.0/2.1中`dofile`和`require`这两个函数现在都是以C函数来实现的，如果被`dofile`或者`require`加载的文件的顶级作用域中调用了`ngx.location.capture*`，`ngx.exec`，`ngx.exit`或者其他的API方法，那么就会触发`"attempt to yield across C-call boundary"`的错误。为了避免这个错误，把这些调用放到你自己的Lua函数里面去，而不是放到顶级的作用域里面。
* 标准Lua 5.1的执行引擎不是完全可恢复的，方法`ngx.location.capture`、`ngx.location.capture_multi`、`ngx.redirect`、`ngx.exec`、`ngx.exit`不能被用在`pcall`或者`xpcall`的语境里面，或者`for ... in ...`语法的第一行，否则会产生`attempt to yield across metamethod/C-call boundary`的错误。请使用LuaJIT 2.x,为了避免这个，它提供了一个完全可恢复的引擎。

### Lua Variable Scope
注意，在导入模块并且使用的时候必须小心

```
local xxx = require('xxx')
```

代替老的废弃的表达式：

```
require('xxx')
```

废弃的原因：按照设计，全局环境拥有准确相同的生存时间作为与它相关联的Nginx的请求处理程序。每个请求处理程序都有自己的Lua全局变量集，这个是请求隔离的主意。Lua模块实际上由第一个Nginx的请求处理程序加载，以及由`package.loaded`内置的`require()`缓存供之后的程序调用。内置的`module()`方法被一些Lua模块所使用，但是在将一个全局变设置到已加载的模块表的时候它有副作用。这个全局变量在请求成立程序结束的时候会被清除掉，每个子请求的处理程序都有自己的（干净的）全局环境。因此访问nil值将会触发Lua的异常。

通常，在ngx_lua的语境中使用Lua全局变量十个非常非常枣糕的主意：
1. 当这些变量其实只是本地变量的时候，在并发请求时将他们设置为全局变量有非常严重的副作用；
2. Lua全局变量需要在全局环境中查找Lua的符号表（只是一个Lua的表），这个成本有点高
3. 一些Lua的全局变量的引用仅仅是输入错误，很难调试。

重要提示：在合理的作用域内应该通过`local`声明变量

可以通过使用[lua-releng](https://github.com/openresty/nginx-devel-utils/blob/master/lua-releng)工具找出来找出你的.lua文件中使用的全局变量：

```
$ lua-releng
Checking use of Lua global variables in file lib/foo/bar.lua ...
        1       [1489]  SETGLOBAL       7 -1    ; contains
        55      [1506]  GETGLOBAL       7 -3    ; setvar
        3       [1545]  GETGLOBAL       3 -4    ; varexpand
```

输出结果说明了在`lib/foo/bar.lua`文件的第1489行向一个名为`contains`的全局变量设置了值，第1506行读取了全局变量`setvar`的值，1545行读取了全局变量`varexpand`。

这个工具将保证Lua模块方法中的局部变量全部都用关键词local声明了，否则将引发运行时的异常。它可以在访问这些变量时候防止不必须的竞争条件的产生。对于背后的原因，请参阅[Data Sharing within an Nginx Worker](http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker)。

### Locations Configured by Subrequest Directives of Other Modules
[ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture)和[ngx.location.capture_multi](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture_multi)指令不能捕获包含[echo_location](http://wiki.nginx.org/HttpEchoModule#echo_location), [echo_location_async](http://wiki.nginx.org/HttpEchoModule#echo_location_async), [echo_subrequest](http://wiki.nginx.org/HttpEchoModule#echo_subrequest), 或者[echo_subrequest_async](http://wiki.nginx.org/HttpEchoModule#echo_subrequest_async)指定的location。

```
location /foo {
	content_by_lua '
		res = ngx.location.capture("/bar")
    ';
}
location /bar {
	echo_location /blah;
}
location /blah {
	echo "Success!";
}
```

```
$ curl -i http://example.com/foo
```

上面的不会运行。

### Special PCRE Sequences
像\d，\s或者\w的PCRE序列，由于在字符串文本中，所以需要特别的注意，反斜杠字符，\，在Lua的语法分析以及Nginx的配置文件解析之前会被过滤掉。所以下面的代码片段无法预期工作：

```
# nginx.conf
? location /test {
?     content_by_lua '
?         local regex = "\d+"  -- THIS IS WRONG!!
?         local m = ngx.re.match("hello, 1234", regex)
?         if m then ngx.say(m[0]) else ngx.say("not matched!") end
?     ';
? }
# evaluates to "not matched!"
```

为了避免这个错误，请使用双反斜杠转义：

```
# nginx.conf
location /test {
	content_by_lua '
		local regex = "\\\\d+"
		local m = ngx.re.match("hello, 1234", regex)
		if m then ngx.say(m[0]) else ngx.say("not matched!") end
	';
}
# evaluates to "1234"
```

这里`\\\\d+`会被Nginx配置文件解析并过滤成`\\d+`，在代码被运行之前会被Lua语言解析器转义成`\d+`。

另外，正则表达式模式可以被提取出来作为一个长的Lua字符串文本并将它包含在长方括号中，[[...]]，在这种情况下，反斜杠只会被Nginx的配置文件解析器转义一次。

```
# nginx.conf
location /test {
	content_by_lua '
		local regex = [[\\d+]]
        local m = ngx.re.match("hello, 1234", regex)
        if m then ngx.say(m[0]) else ngx.say("not matched!") end
	';
}
# evaluates to "1234"
```

这里的`[[\\d+]]`将被Nginx的配置文件解析器正确的转换成`[[\d+]]`。

注意，有一种更长的长括号的表达式，[=[...]=]，如果正则表达式模式包含[...]序列它可能会被用上。如果需要，[=[...]=]可以用作默认表达式。

```
# nginx.conf
location /test {
	content_by_lua '
    	local regex = [=[[0-9]+]=]
        local m = ngx.re.match("hello, 1234", regex)
        if m then ngx.say(m[0]) else ngx.say("not matched!") end
    ';
}
# evaluates to "1234"
```

另一个避免PCRE序列被转义的方法是确保Lua代码被放在一个另外的脚本文件中，并使用各种的*_by_lua_file指令来执行它。使用这个方法，反斜杠只会被Lua语法解析器转义一次，整个过程仅仅在这里被转义一次。

```
-- test.lua
local regex = "\\d+"
local m = ngx.re.match("hello, 1234", regex)
if m then ngx.say(m[0]) else ngx.say("not matched!") end
-- evaluates to "1234"
```

在外部脚本文件中，PCRE序列目前作为一个长括号的Lua字符串文本而不需要被修改。

```
-- test.lua
local regex = [[\d+]]
local m = ngx.re.match("hello, 1234", regex)
if m then ngx.say(m[0]) else ngx.say("not matched!") end
-- evaluates to "1234"
```

### Mixing with SSI Not Supported
Mixing SSI with ngx_lua in the same Nginx request is not supported at all. Just use ngx_lua exclusively. Everything you can do with SSI can be done atop ngx_lua anyway and it can be more efficient when using ngx_lua.

### SPDY Mode Not Fully Supported
Certain Lua APIs provided by ngx_lua do not work in Nginx's SPDY mode yet: ngx.location.capture, ngx.location.capture_multi, and ngx.req.socket.

## TODO

### Short Term
* review and apply Jader H. Silva's patch for ngx.re.split().
* review and apply vadim-pavlov's patch for ngx.location.capture's extra_headers option
* use ngx_hash_t to optimize the built-in header look-up process for ngx.req.set_header, ngx.header.HEADER, and etc.
* add configure options for different strategies of handling the cosocket connection exceeding in the pools.
* add directives to run Lua codes when nginx stops.
* add ignore_resp_headers, ignore_resp_body, and ignore_resp options to ngx.location.capture and ngx.location.capture_multi methods, to allow micro performance tuning on the user side.

### Longer Term
* add automatic Lua code time slicing support by yielding and resuming the Lua VM actively via Lua's debug hooks.
* add stat mode similar to mod_lua.

## Changes
The changes of every release of this module can be obtained from the ngx_openresty bundle's change logs:
[http://openresty.org/#Changes](http://openresty.org/#Changes)

## Test Suite
The following dependencies are required to run the test suite:
* Nginx version >= 1.4.2
* Perl modules:
	* Test::Nginx: [http://github.com/openresty/test-nginx](http://github.com/openresty/test-nginx)
* Nginx modules:
	* [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit)
	* [ngx_set_misc](http://github.com/openresty/set-misc-nginx-module)
	* [ngx_auth_request]()
	* [](http://mdounin.ru/files/ngx_http_auth_request_module-0.2.tar.gz)(this is not needed if you're using Nginx 1.5.4+.)
	* [ngx_echo](http://github.com/openresty/echo-nginx-module)
	* [ngx_memc](http://github.com/openresty/memc-nginx-module)
	* [ngx_srcache](http://github.com/openresty/srcache-nginx-module)
	* ngx_lua (i.e., this module)
	* [ngx_lua_upstream](http://github.com/openresty/lua-upstream-nginx-module)
	* [ngx_headers_more](http://github.com/openresty/headers-more-nginx-module)
	* [ngx_drizzle](http://github.com/openresty/drizzle-nginx-module)
	* [ngx_rds_json](http://github.com/openresty/rds-json-nginx-module)
	* [ngx_coolkit](https://github.com/FRiCKLE/ngx_coolkit)
	* [ngx_redis2](http://github.com/openresty/redis2-nginx-module)

The order in which these modules are added during configuration is important because the position of any filter module in the filtering chain determines the final output, for example. The correct adding order is shown above.

* 3rd-party Lua libraries:
	* [lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php)
* Applications:
	* mysql: create database 'ngx_test', grant all privileges to user 'ngx_test', password is 'ngx_test'
	* memcached: listening on the default port, 11211.
	* redis: listening on the default port, 6379.

See also the [developer build script](https://github.com/openresty/lua-nginx-module/blob/master/util/build2.sh) for more details on setting up the testing environment.

To run the whole test suite in the default testing mode:

```
cd /path/to/lua-nginx-module
export PATH=/path/to/your/nginx/sbin:$PATH
prove -I/path/to/test-nginx/lib -r t
```

To run specific test files:

```
cd /path/to/lua-nginx-module
export PATH=/path/to/your/nginx/sbin:$PATH
prove -I/path/to/test-nginx/lib t/002-content.t t/003-errors.t
```

To run a specific test block in a particular test file, add the line --- ONLY to the test block you want to run, and then use the `prove` utility to run that .t file.

There are also various testing modes based on mockeagain, valgrind, and etc. Refer to the [Test::Nginx documentation](http://search.cpan.org/perldoc?Test::Nginx) for more details for various advanced testing modes. See also the test reports for the Nginx test cluster running on Amazon EC2: [http://qa.openresty.org](http://qa.openresty.org/).

## Copyright and License
This module is licensed under the BSD license.

Copyright (C) 2009-2014, by Xiaozhe Wang (chaoslawful) <chaoslawful@gmail.com>.

Copyright (C) 2009-2014, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:
* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

## See Also

* [lua-resty-memcached](https://github.com/openresty/lua-resty-memcached) library based on ngx_lua cosocket.
* [lua-resty-redis](https://github.com/openresty/lua-resty-redis) library based on ngx_lua cosocket.
* [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql) library based on ngx_lua cosocket.
* [lua-resty-upload](https://github.com/openresty/lua-resty-upload) library based on ngx_lua cosocket.
* [lua-resty-dns](https://github.com/openresty/lua-resty-dns) library based on ngx_lua cosocket.
* [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket) library for both WebSocket server and client, based on ngx_lua cosocket.
* [lua-resty-string](https://github.com/openresty/lua-resty-string) library based on LuaJIT FFI.
* [lua-resty-lock](https://github.com/openresty/lua-resty-lock) library for a nonblocking simple lock API.
* [lua-resty-cookie](https://github.com/cloudflare/lua-resty-cookie) library for HTTP cookie manipulation.
* [Routing requests to different MySQL queries based on URI arguments](https://openresty.org/#RoutingMySQLQueriesBasedOnURIArgs)
* [Dynamic Routing Based on Redis and Lua](https://openresty.org/#DynamicRoutingBasedOnRedis)
* [Using LuaRocks with ngx_lua](https://openresty.org/#UsingLuaRocks)
* [Introduction to ngx_lua](https://github.com/openresty/lua-nginx-module/wiki/Introduction)
* [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit)
* [HttpEchoModule](http://wiki.nginx.org/HttpEchoModule)
* [HttpDrizzleModule](http://wiki.nginx.org/HttpDrizzleModule)
* [postgres-nginx-module](http://github.com/FRiCKLE/ngx_postgres)
* [HttpMemcModule](http://wiki.nginx.org/HttpMemcModule)
* [The ngx_openresty bundle](http://openresty.org/)
* [Nginx Systemtap Toolkit](https://github.com/openresty/nginx-systemtap-toolkit)

## Directives

### lua_use_default_type
syntax： `lua_use_default_type on | off`
default: `lua_use_default_type on`
context: `http, server, location, location if`

指定是否使用由指令`default_type`指定的响应头`Content-Type`的默认MIME值。如果你不想在你的Lua请求处理程序中为`Content-Type`设置默认值，那么请将这个指令设置为关闭状态。

这个指令默认为打开。

在 v0.9.1 版本中首先介绍了这个指令。


### lua_code_cache
syntax: `lua_code_cache on | off`
default: `lua_code_cache on`
context: `http, server, location, location if`

在`*_by_lua_file`（例如：set_by_lua_file、content_by_lua_file）指令和Lua模块中为Lua代码开启或者关闭代码缓存功能。

当关闭的时候，从 v0.9.3版开始每个由ngx_lua处理的请求将运行在一个单独的Lua VM实例上。由`set_by_lua_file`、`content_by_lua_file`、`access_by_lua_file`等引入的文件将不会被缓存以及所有的Lua模块将从零开始都会重新加载。在这个地方，开发人员边可以编辑完刷新就可以看到效果。

不过请注意，通过指令`set_by_lua`,`content_by_lua`,`access_by_lua`,`rewrite_by_lua`写到nginx.conf中的Lua代码在你修改完后不会马上看到效果，因为nginx.conf只会被Nginx配置文件解析器解析，要想刷新看到效果只能给Nginx发送`HUP`信号或者重启Nginx。

即使启用代码缓存，`*_by_lua_file`加载的Lua文件中通过`dofile`或者`loadfile`再次加载的Lua文件不会被缓存（除非你自己做相关缓存的工作）。通常的，你也可以使用`init_by_lua`或`init_by_lua_file`指令来加载所有此类文件或者这些Lua文件包含的真实的Lua模块，然后通过`require`来加载。

ngx_lua模块不支持stat模式（至今为止）。

强烈建议不要在生产环境中禁用Lua代码缓存，这个功能仅仅在开发过程中使用，因为它对整体性能有很大的影响。例如，在禁用Lua代码缓存前后，输出一个“hello world”的性能会有一个数量级的差距。


### lua_regex_cache_max_entries
syntax: `lua_regex_cache_max_entries <num>`
default: `lua_regex_cache_max_entries 1024`
context: `http`

指定允许在工作进程级编译的正则缓存的最大数量的条目。

如果正则表达式选项`o`（只编译一次标志）被指定了，在`ngx.re.match`、`ngx.re.gmatch`、`ngx.re.sub`、`ngx.re.gsub`中使用的正则表达式将会被缓存在此缓存中。

默认能缓存的条目数量是1024，当缓存的数目达到这个限制后，新正则表达式将不再被缓存（如果未指定o选项），然后`error.log`将会写入一条（仅仅一条）警告信息：

```
2011/08/27 23:18:26 [warn] 31997#0: *1 lua exceeding regex cache max entries (1024), ...
```

为了避免碰撞到指定的限制，请不要激活用于正则表达式（`ngx.re.sub`和`ngx.re.gsub`的/或者`replace`字符串参）的o选项，动态生成会引起无限的变化。
Do not activate the o option for regular expressions (and/or replace string arguments for ngx.re.sub and ngx.re.gsub) that are generated on the fly and give rise to infinite variations to avoid hitting the specified limit.


### lua_regex_match_limit
syntax: `lua_regex_match_limit <num>`
default: `lua_regex_match_limit 0`
context: `http`

执行ngx.re API时候用于指定使用PCRE库的“匹配限制”。引用PCRE手册，“the limit ... has the effect of limiting the amount of backtracking that can take place.”

命中限制时,ngx.re API函数会凡是错误`pcre_exec() failed: -8`

当把限制设置为0时，默认的“匹配限制”会在编译PCRE库时候被使用。0是这个指令的默认值。

在 v0.8.5 版本中首先介绍了这项指令。


### lua_package_path
syntax: `lua_package_path <lua-style-path-str>`
default: `环境变量LUA_PATH的值或者编译于Lua中的默认值`
context: `http`

设置`set_by_lua`、`content_by_lua`等指令使用的Lua模块搜索路径。路径字符串是标准的Lua路径形式，;;可以用来替代原始的搜索路径。

从v0.5.0rc29版本开始，特殊符号`$prefix`或者`${prefix}`可以用在搜索路径字符串表示`server prefix`的路径，`server prefix`通常在启动Nginx时候通过命令`-p PATH`指定。


### lua_package_cpath
syntax: `lua_package_cpath <lua-style-cpath-str>`
default: `环境变量LUA_CPATH的值或者编译于Lua中的默认值`
context: `http`

设置`set_by_lua`、`content_by_lua`等指令使用的Lua C模块搜索路径。路径字符串是标准的Lua路径形式，;;可以用来替代原始的搜索路径。

从v0.5.0rc29版本开始，特殊符号`$prefix`或者`${prefix}`可以用在搜索路径字符串表示`server prefix`的路径，`server prefix`通常在启动Nginx时候通过命令`-p PATH`指定。


### init_by_lua
syntax: `init_by_lua <lua-script-str>`
context: `http`
phase: `loading-config`

当Nginx的master进程加载Nginx配置文件时候，在全局Lua引擎级别运行由参数`<lua-script-str>`指定的Lua代码。

当Nginx接收到`HUP`信号并开始重新加载配文件时候，Lua引擎将会重新创建，`init_by_lua`指令将在新的Lua引擎中再次运行。在`lua_code_cache`指令关闭的情况下（默认开启），`init_by_lua`处理程序将会在每个请求上运行，因为在此特殊模式下，总是为每个请求创建一个独立的Lua VM。

通常，你可以通过这个钩子在服务器启动时候注册（真实的）Lua全局变量或者预加载Lua模块。下面是一个预加载Lua模块的例子：

```
init_by_lua 'cjson = require "cjson"';
 
server {
	location = /api {
    	content_by_lua '
        	ngx.say(cjson.encode({dog = 5, cat = 6}))
        ';
    }
}
```

你也可以在这个阶段初始化[`lua_shared_dict`](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict)共享内存存储。下面是例子：

```
lua_shared_dict dogs 1m;
 
init_by_lua '
	local dogs = ngx.shared.dogs;
    dogs:set("Tom", 56)
';
 
server {
	location = /api {
    	content_by_lua '
        	local dogs = ngx.shared.dogs;
            ngx.say(dogs:get("Tom"))
        ';
    }
}
```

但请注意，[`lua_shared_dict`](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict)的共享内存存储在配置重新加载（例如：通过`HUP`信号）的时候不会被清除。所以，在这种情况下如果在你的`init_by_lua`代码中不想重新初始化共享内存存储，你只需要在共享内存存储中设置一个自定义的标志，并经常在`init_by_code`中检查。

这种语境中的Lua代码会在Nginx fork他的工作进程前运行，被加载的数据或者代码在所有的工作进程间将享受由操作系统提供的写时复制（COW）功能，从而节省大量的内存。

在这个语境中不要初始化你自己的Lua全局变量，因为Lua全局变量的使用会降低性能，并可能到导致全局命名空间污染（有关更多详细信息请参阅[Lua变量范围](http://wiki.nginx.org/HttpLuaModule#Lua_Variable_Scope)）。推荐的方法是使用适当的Lua模块文件（不要使用标准的Lua函数`module()`来定义Lua模块，因为它会污染全局命名空间），在`init_by_lua`或者其他语境中使用`require()`来加载你自己的模块文件（`require()`会将加载的Lua模块缓存到全局`package.loaded`表中，因此你的模块在整个Lua VM实例中仅仅只会加载一次）。

在这方面N[ginx API为Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)提供了一小整套支持：
* 日志APIs: [ngx.log](http://wiki.nginx.org/HttpLuaModule#ngx.log) and [print](http://wiki.nginx.org/HttpLuaModule#print),
* 共享字典API：[ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT)

在这方面的未来用户请求上，将为Lua提供更多的Nginx API。

基本上你可以在这种情况下安全的使用Lua库进行阻塞I/O操作，因为在服务启动时候阻塞主进程是完全没关系的。即便是在Nginx内核在加载配置阶段做阻塞I/O操作。

在这个语境中，你需要对你的Lua代码注册方面存在的潜在安全漏洞非常小心，因为Nginx的主进程通常都是以root权限在运行。

在 v0.5.5 版本中首先介绍了这项指令。


### init_by_lua_file
syntax: `init_by_lua_file <path-to-lua-script-file>`
context: `http`
phase: `loading-config`

相当于`init_by_lua`,只是`<path-to-lua-script-file>`指向包含Lua代码或者Lua/LuaJIT的字节码的文件。

如果给出的是一个类似`foo/bar.lua`的相对路径，那么服务器会默认到由Nginx启动时候添加的命令行参数`-p PATH`指定的`server prefix`路径下去寻找文件。

在 v0.5.5 版本中首先介绍了这项指令。


### init_worker_by_lua
syntax: `init_worker_by_lua <lua-script-str>`
context: `http`
phase: `starting-worker`

当启用主进程时候，在每个Nginx工作进程启动时候运行指定的Lua代码。当主进程处于禁用状态时，这个钩子仅仅在`init_by_lua*`后运行。

这个钩子经常被用来创建每个工作进程的重复定时器（使用`ngx.timer.at` API），无论是后端运行状况检查或者其他日常计时工作。例如：

```
init_worker_by_lua '
	local delay = 3  -- in seconds
    local new_timer = ngx.timer.at
    local log = ngx.log
    local ERR = ngx.ERR
    local check
 
    check = function(premature)
    	if not premature then
        	-- do the health check or other routine work
            local ok, err = new_timer(delay, check)
            if not ok then
            	log(ERR, "failed to create timer: ", err)
                return
            end
        end
	end
 
	local ok, err = new_timer(delay, check)
    if not ok then
    	log(ERR, "failed to create timer: ", err)
        return
    end
';
```

这项指令是首先介绍了在 v0.9.5 版本中


### init_worker_by_lua_file
syntax: `init_worker_by_lua_file <lua-file-path>`
context: `http`
phase: `starting-worker`

类似于`init_worker_by_lua`，但是接收的参数是一个Lua源文件或者Lua字节码文件的路径。

在 v0.9.5 版本中首先介绍了这项指令。


### set_by_lua
syntax: `set_by_lua $res <lua-script-str> [$arg1 $arg2 ...]`
context: `server, server if, location, location if`
phase: `rewrite`

执行由`<lua-script-str>`指定的代码，接收传入的参数`$arg1 $arg2 ...`，并将返回的字符串赋值给`$res`。`<lua-script-str>`中的Lua代码可以进行API的调用，并可以从`ngx.arg`表（索引从1开始按序增加）中访问输入的参数。

这个指令旨在执行时间段，运行速度快的代码块，因为在代码执行期间Nginx的时间轮询会被阻塞。因而需要避免非常耗时的代码。

这个指令用于注入用户指令到标准的[`HttpRewriteModule`](http://wiki.nginx.org/HttpRewriteModule)命令列表里面。因为在[`HttpRewriteModule`](http://wiki.nginx.org/HttpRewriteModule)的命令列表里面不支持非阻塞I/O的操作，Lua APIs requiring yielding the current Lua "light thread" cannot work in this directive。

至少下面的API在`set_by_lua`中是被禁用的：
* 输出API方法（例如：`ngx.say`和`ngx.send_headers`）
* 控制API方法（例如：`ngx.exit`）
* 子请求API方法（例如：`ngx.location.capture`和`ngx.location.capture_multi`）
* Cosocket API方法（例如：`ngx.socket.tcp`和`ngx.req.socket`）
* 休眠API方法`ngx.sleep`

此外，请注意这个指令每次只能将值赋给一个Nginx的变量。然而，可以用`ngx.var.VARIABLE`替代。

```
location /foo {
	set $diff ''; # we have to predefine the $diff variable here
 
    set_by_lua $sum '
    	local a = 32
        local b = 56
 
        ngx.var.diff = a - b;  -- write to $diff directly
        return a + b;          -- return the $sum value normally
    ';
 
    echo "sum = $sum, diff = $diff";
}
```

这个指令可以自有的与[`HttpRewriteModule`](http://wiki.nginx.org/HttpRewriteModule)、[`HttpSetMiscModule`](http://wiki.nginx.org/HttpSetMiscModule)、[`HttpArrayVarModule`](http://wiki.nginx.org/HttpArrayVarModule)模块搜有的指令混合使用。所有的这些指令将按照他们在配置文件中出现的顺序依次运行。

```
set $foo 32;
set_by_lua $bar 'tonumber(ngx.var.foo) + 1';
set $baz "bar: $bar";  # $baz == "bar: 33"
```

As from the v0.5.0rc29 release, Nginx variable interpolation is disabled in the `<lua-script-str> `argument of this directive and therefore, the dollar sign character ($) can be used directly.

这个指令需要`ngx_devel_kit`模块。


### set_by_lua_file
syntax: `set_by_lua_file $res <path-to-lua-script-file> [$arg1 $arg2 ...]`
context: `server, server if, location, location if`
phase: `rewrite`

相当于`set_by_lua`，只是内容换成了由`<path-to-lua-script-file>`指定的Lua文件包含的代码。从 v0.5.0rc32 版本开始，指定的Lua脚本可以是Lua/LuaJIT的字节码了。

Nginx内部变量可以在`<path-to-lua-script-file>`指定的文件中使用。但是要特别注意注入式攻击。

当给的lua文件的路径是类似`foo/bar.lua`的相对路径时候，那么将按照相对于路径`server prefix`进行查找，这个路径在Nginx启动时候由参数`-p`指定。

当开启Lua代码缓存的时候（默认开启），用户代码在第一次请求的时候被加载并缓存，并且只加载一次，当Lua源码被修改了必须要重新加载Nginx的配置文件才能起效。在进行开发的时候，在`nginx.conf`文件中使用` lua_code_cache off`关闭代码缓存可以避免每次修改都要Nginx重新加载配置文件。

这项指令要求 ngx_devel_kit 模块


### content_by_lua
syntax: `content_by_lua <lua-script-str>`
context: `location, location if`
phase: `content`

作为内容处理程序，在每次请求中执行由`<lua-script-str>`指定的Lua代码。Lua代码可能会进行API调用以及在一个独立的全局环境中生成一个新的协程执行(i.e. a sandbox)。

不要再同一个location中将这个指令和其他的内容处理指令一起使用。例如，此指令和proxy_pass不能再同一个location同时出现。


### content_by_lua_file
syntax: `content_by_lua_file <path-to-lua-script-file>`
context: `location, location if`
phase: `content`

相当于`content_by_lua`，只是由`<path-to-lua-script-file>`指定的文件包含Lua代码，或从 v0.5.0rc32 版本开始计入的Lua/LuaJIT字节码。

可以在`<path-to-lua-script-file>`指定的文件中使用Nginx变量。这个有一定的风险，通常不建议这样使用。

当给的lua文件的路径是类似`foo/bar.lua`的相对路径时候，那么将按照相对于路径`server prefix`进行查找，这个路径在Nginx启动时候由参数`-p`指定。

当开启Lua代码缓存的时候（默认开启），用户代码在第一次请求的时候被加载并缓存，并且只加载一次，当Lua源码被修改了必须要重新加载Nginx的配置文件才能起效。在进行开发的时候，在`nginx.conf`文件中使用` lua_code_cache off`关闭代码缓存可以避免每次修改都要Nginx重新加载配置文件。


### rewrite_by_lua
syntax: `rewrite_by_lua <lua-script-str>`
context: `http, server, location, location if`
phase: `rewrite tail`

作为重写阶段的处理程序，在每个请求中运行由`<lua-script-str>`指定的Lua代码。Lua代码可能会进行API调用以及在一个独立的全局环境中生成一个新的协程执行(i.e. a sandbox)。

请注意，此处理程序始终运行在标准的`HttpRewriteModule`之后。因此，以下将按预期方式工作：

```
location /foo {
	set $a 12; # create and initialize $a
    set $b ""; # create and initialize $b
    rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
    echo "res = $b";
}
```

`set $a 12`以及`set $b ""`在`rewrite_by_lua`之前运行。

另一方面，以下将无法按预期工作：

```
location /foo {
	set $a 12; # create and initialize $a
	set $b ''; # create and initialize $b
	rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
	if ($b = '13') {
		rewrite ^ /bar redirect;
		break;
	}

	echo "res = $b";
}
```

因为`if`在`rewrite_by_lua`之前运行，既是在配置中它被放在后面。

正确的方式如下：

```
location /foo {
	set $a 12; # create and initialize $a
    set $b ''; # create and initialize $b
    rewrite_by_lua '
    	ngx.var.b = tonumber(ngx.var.a) + 1
        if tonumber(ngx.var.b) == 13 then
        	return ngx.redirect("/bar");
        end
    ';
 
    echo "res = $b";
}
```

注意，[`ngx_eval`](http://www.grid.net.ru/nginx/eval.en.html)模块近似于使用`rewrite_by_lua`。例如：

```
location / {
	eval $res {
    	proxy_pass http://foo.com/check-spam;
    }
 
    if ($res = 'spam') {
    	rewrite ^ /terms-of-use.html redirect;
    }
 
    fastcgi_pass ...;
}
```

在ngx_lua中的实现方式如下：

```
location = /check-spam {
	internal;
    proxy_pass http://foo.com/check-spam;
}
 
location / {
	rewrite_by_lua '
    	local res = ngx.location.capture("/check-spam")
        if res.body == "spam" then
        	return ngx.redirect("/terms-of-use.html")
        end
    ';
 
    fastcgi_pass ...;
}
```

跟其他重写阶段程序一样，`rewrite_by_lua`也运行于子请求中。

注意，在`rewrite_by_lua`处理程序中调用`ngx.exit(ngx.OK)`时候，nginx的请求处理控制流让然会继续进入内容处理程序中。在`rewrite_by_lua`里面中止当前的请求，调用`ngx.exit`并传入大于等于200(ngx.HTTP_OK)以及小于300（ngx.HTTP_SPECIAL_RESPONSE）的状态码就可以成功的退出，使用`ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)`(或者其他的错误码)以错误形式退出。

如果HttpRewriteModule模块的的重写指令用来更改URI和启动位置重新查找（内部重定向），那么任何处于当前location中的`rewrite_by_lua`或者`rewrite_by_lua_file`的代码将不会被执行。例如：

```
location /foo {
	rewrite ^ /bar;
    rewrite_by_lua 'ngx.exit(503)';
}
location /bar {
    ...
}
```

这里的Lua代码`ngx.exit(503)`永远都不会被运行。如果这里使用`rewrite ^ /bar last`，这也同样的会启动一个内部重定向。如果改用了`break`修饰符来替代，将不会产生内部重定向并且`rewrite_by_lua`的代码将会被运行。

`rewrite_by_lua`的代码总是会在`rewrite`请求处理阶段之后运行，除非`rewrite_by_lua_no_postpone`处于开启状态。


### rewrite_by_lua_file
syntax: `rewrite_by_lua_file <path-to-lua-script-file>`
context: `http, server, location, location if`
phase: `rewrite tail`

相当于`rewrite_by_lua`，只是由`<path-to-lua-script-file>`指定的文件包含Lua代码，或从 v0.5.0rc32 版本开始加入的Lua/LuaJIT字节码。

可以在`<path-to-lua-script-file>`指定的文件中使用Nginx变量。这个有一定的风险，通常不建议这样使用。

当给的lua文件的路径是类似`foo/bar.lua`的相对路径时候，那么将按照相对于路径`server prefix`进行查找，这个路径在Nginx启动时候由参数`-p`指定。

当开启Lua代码缓存的时候（默认开启），用户代码在第一次请求的时候被加载并缓存，并且只加载一次，当Lua源码被修改了必须要重新加载Nginx的配置文件才能起效。在进行开发的时候，在`nginx.conf`文件中使用` lua_code_cache off`关闭代码缓存可以避免每次修改都要Nginx重新加载配置文件。

`rewrite_by_lua_file`总是在`rewrite`请求处理阶段结束时运行，除非开启了`rewrite_by_lua_no_postpone`。


### access_by_lua
syntax: `access_by_lua <lua-script-str>`
context: `http, server, location, location if`
phase: `access tail`

