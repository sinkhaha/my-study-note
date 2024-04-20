# Node.js 简介

`Node.js` 是一个开源的、跨平台的、基于 `V8` 引擎 的 `JavaScript` 运行时环境。`Node.js` 使得我们可以使用 ` JavaScript` 编写服务端应用。其跟 `JavaScript` 的关系如下图：



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/Node-javascript%E7%9A%84%E5%85%B3%E7%B3%BB.png)

运行时是指运行时环境，可以理解为 `JavaScript` 语言的宿主环境，就是程序在运行期间需要依赖的一系列组件库。`JavaScript` 是一门实现了 `ECMAScript` 语法规范的语言，提供了一些内置对象和 `API`，但 `JavaScript` 不提供网络、文件、进程等功能，这些功能可由运行时环境实现和提供，比如 `Node.js`。



# Node.js 的组成

`Node.js` 底层主要由 `V8` 引擎、`libuv` 库 和 一些 `第三方C/C++库` 组成。`Node.js` 依赖于这些底层库，封装提供了一些内置的核心模块，以供给 `JavaScript` 环境使用。



## V8 引擎

`V8` 引擎是一个用 `C++` 编写的高性能的 `JavaScript` 引擎，它负责解析和执行 `JavaScript` 代码，同时也提供了内存管理、垃圾回收等功能。`V8` 的高性能主要体现在，`V8` 引擎将 `JavaScript` 代码直接编译成机器码，而不是字节码，进而提高了 `V8` 在执行 `JavaScript` 的效率；并且使用了缓存机制来提高性能。



## libuv

`libuv` 是一个跨平台的、基于事件驱动 的 异步 `I/O` 库，它提供了以下功能：

* 事件循环

* 网络通信

* 文件系统操作

* 异步文件

* 进程管理

* 线程池

* 信号处理

* 定时器

  

**主要有两个重点概念：事件驱动 和 异步 I/O**

1. 事件驱动：`libuv` 为 `Node.js` 提供了事件循环机制，它本质是一个生产/消费模型；在事件循环中每个阶段会维护自己的任务队列，事件循环过程中会消费这些任务，我们也可以往任务队列生产新的任务，驱动着`libuv`

   > 更多关于事件循环的讲解可参考[这篇文章](https://zhuanlan.zhihu.com/p/622381734)

2. 异步 `I/O`：`libuv` 的异步机制依赖于不同平台的异步机制（比如 `Linux` 的 `epoll` ，`macOS` 的 `kqueue`, `Windows `的 `IOCP`）。它是非阻塞 `I/O` ，即当调用一个系统调用时不会阻塞线程；比如读取文件，如果数据没有准备好，线程不会被阻塞，可以继续去做另外的事情，等到数据准备好了，操作系统会通过事件驱动通知我们，然后在事件循环中会处理该事件，进而处理读到的数据

   

`Node.js` 就是依赖 `libuv` 实现的，这也决定了 `Node.js` 的特点：`Node.js` 是基于事件的、单线程的异步 `I/O` 架构；所以 `Node.js` 更适合处理 `I/O` 密集型任务，不适合处理 `CPU` 密集型任务，避免阻塞事件循环



## 其他第三方 C/C++ 库

`Node.js` 还集成了很多高性能的 `C/C++ `开源库，比如：

* `http-parser` 或 `llhttp`：是一个用 `C` 实现的 `http` 消息解析器，用于解析 `http` 的请求数据和返回数据（ `http-parser` 是老版本的依赖，`Node.js v12` 版本后使用 `llhttp`）

* `OpenSSL`：是一个用 `C` 实现的是跨平台的安全套接字层协议库，实现了加密功能：`SSL` 与 `TLS` 协议
* `zlib`：是一个数据的压缩和解压库
* `c-ares`：是一个异步 `DNS` 查询和解析库



**`Node.js` 的组成如下图：**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/Node.js%20%E7%9A%84%E7%BB%84%E6%88%90.drawio.png)



由图可知， `Bindings` 和 `Addon` 是连接 `JavaScript` 代码和 `C/C++` 代码的桥梁

* `Bindings`：在 `Node.js` 中，`Bindings` 就是一些胶水代码，把 `JavaScript `和 `C/C++` 绑定在一起，使它们能够互相调用；通过 `Bindings` 可以把 `Node.js` 中用到 `C/C++` 的库的接口暴露给 `JavaScript` 环境

* `C++ Addons`：基于 `V8`  的可扩展能力，我们也可以把自己的 `C/C++库` 封装成 `C++` 扩展模块，使接口其能够暴露给 `JavaScript` 环境，这就是 `C++ Addon`

  > 更多关于 `C++ Addons` 的讲解可参考[这篇文章](https://zhuanlan.zhihu.com/p/584943566)
  
  

# 参考

* [libuv](http://docs.libuv.org/en/v1.x/design.html)
* [Node.js](https://nodejs.org/en)
* [Node.js是个啥](https://juejin.cn/book/7196627546253819916/section/7195089399787290635)
* [Node.js Dependencies](https://nodejs.org/en/docs/meta/topics/dependencies#llhttp)
