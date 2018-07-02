# 一张脑图说清 Nginx 的主流程

这个脑图在 [nginx-1.14.0-research](https://github.com/its-tech/nginx-1.14.0-research/tree/master/docs) 上。这是我在研究nginx的http模块的时候画的。基本上把 Nginx 主流程（特别是 HTTP 的部分）的关键函数和关键设置画了下来，了解了这个脑图，就对整个 Nginx 的主流程有了定性的了解了。

Nginx 的启动过程分为两个部分，一个部分是读取配置文件，做配置文件中配置的一些事情（比如监听端口等）。第二个部分是形成 Master-Worker 的多进程模型。这两个过程就是 Nginx 代码中最重要的两个函数：`ngx_init_cycle` 和 `ngx_master_process_cycle`

![](http://tuchuang.funaio.cn/18-6-29/29766630.jpg)

# ngx_init_cycle

ngx_init_cycle 是 Nginx 中最重要的函数，没有之一。我们可以想想，如果我们写一个和 Nginx 一样的 Web 服务，我们会怎么做？我们大致的思路一定是解析配置文件，把配置文件存入到一个数据结构中，然后根据数据结构，进行端口监听。是的，差不多，Nginx 就是这么一个流程。不过 Nginx 里面有个模块的概念，所有的功能都是用模块的方式进行加载的。

## Nginx 的模块

Nginx 的模块分为几类，这几类分别为 Core，Event，Conf，Http，Mail。看名字就知道 Core 模块是最重要的。模块是什么意思呢？它包含一堆命令（cmd）和命令对应的处理函数（cmd->handler），我们根据配置文件中的配置（token）就知道这个配置是属于哪个模块的哪个命令，然后调用命令对应的处理函数来处理或者设置我们的服务。

这几类模块中，Core 模块是 Nginx 启动的时候一定会加载的，其他的模块，只有在解析配置的时候，遇到了这个模块的命令，才会加载对应的模块。
这个也是体现了 Nginx 按需加载的理念。（昨天还和小组成员讨论，如果我们写的话，可能就会先把所有模块都加载，然后根据配置文件进行匹配，这样可能 Nginx 的启动过程和进程资源就变大了）。

模块的另一个问题是我这个 Nginx 最多有哪些模块的能力呢？这个是编译的时候就决定了，Nginx 的编译过程可以参考这篇[文章](https://www.cnblogs.com/yjf512/p/9177562.html) 。我们可以不用管./configure 的时候的具体内容，但是我们最关注的就是 `objs/ngx_modules.c` 这个编译出来的文件，里面有个`ngx_modules`全局变量，这个变量里面就存放了我们这次编译的 Nginx 最多可以支持的模块。

模块的结构是我们需要关注的另外一个问题。 Nginx 中模块的结构叫做`ngx_module_s`（你或许会看到`ngx_module_t`，其实就是`struct ngx_moudle_s`的缩写）

![](http://tuchuang.funaio.cn/18-6-29/26236212.jpg)

里面有个结构`*ctx`，对于不同的模块类型，这个`ctx`指向的结构是不一样的，我们这里最主要是研究 HTTP 类型的模块，所以我们就记得 HTTP 模块指向的结构是`ngx_http_module_t`

![](http://tuchuang.funaio.cn/18-6-29/79112482.jpg)

## 主流程

了解了 Nginx 的模块概念，我们再回到`ngx_init_cycle`函数

![](http://tuchuang.funaio.cn/18-6-29/63307342.jpg)

这个函数里面做了几个事情:

* `ngx_cycle_modules`，它本质就是把`objs/ngx_modules.c`里面的全局变量拷贝到`cycle`这个全局变量里面
* 调用了每个 Core 类型模块的`create_conf`方法
* `ngx_conf_parse` 解析配置文件，调用每个Core 类型模块的`init_conf`方法
* 调用了每个 Core 类型模块的`init_conf`方法
* `ngx_open_listening_sockets` 打开配置文件中设置的监听端口和IP
* `ngx_init_modules` 调用每个加载模块的`init_module`方法

`create_conf`是创建一些模块需要初始化的结构，但是这个结构里面并没有具体的值。`init_conf`是往这些初始化结构里面填写配置文件中解析出来的信息。

其中的`ngx_conf_parse`是真正解析配置文件的。

在代码`ngx_open_listening_sockets`里面我们看到熟悉的bind，listen的命令。所以 Nginx 是如何多个进程同时监听一个80端口的？本质是启动了一个master进程，在`ngx_init_cycle`里面监听了端口，然后在`ngx_master_process_cycle`里面 fork 出来多个 worker 子进程。

## ngx_conf_parse

这个函数是非常非常重要的。

![](http://tuchuang.funaio.cn/18-6-29/56409966.jpg)

它的逻辑，就是这两步，首先使用函数`ngx_conf_read_token`先循环逐行逐字符查找，看匹配的字符，获取出`cmd`, 然后去所有的模块查找对应的`cmd`,调用那个查找后的`cmd->set`方法。用Http模块举例子，我们的配置文件中一定有且只有一个关键字叫http
```
http{

}
```
先解析这个配置的时候发现了`http`这个关键字，然后去各个模块匹配，发现`ngx_http_module`这个模块包含了`http`命令。它对应的set方法是`ngx_http_block`。这个方法就是http模块非常重要的方法了。当然，这里顺带提一下，event模块也有类似的方法，`ngx_events_block`。它具体做的事情就是解析
```
event epoll
```
这样的命令，并创建出事件驱动的模型。

## ngx_http_block
