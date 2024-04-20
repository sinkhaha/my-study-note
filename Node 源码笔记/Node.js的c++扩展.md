# 什么是C++扩展

Node.js本身是基于Chrome V8引擎 和 libuv，用C++开发的，所以其底层头文件暴露的API是适用于C++的，使得Node.js能像导入JS模块一样导入C++扩展。



**C++扩展的本质**

一个编译好的C++扩展模块是后缀名为.node的动态链接库，Node.js导入一个C++扩展模块其实就是在Node.js运行时引入一个动态链接库的过程。



**C++扩展的加载**

Node.js使用process.dlopen方法加载.node模块，具体可参考另一篇[文章](https://zhuanlan.zhihu.com/p/582859603)

> process.dlopen实际是封装了C++层src/node_binding.cc文件的DLOpen函数，DLOpen又调用了libuv的uv_dlopen函数用于加载 .node文件



# 为什么要用C++扩展

1. 性能问题

   一个编译好的`C++二进制文件`的执行效率通常比JS解析器解析JS代码高效，比如计算密集型等场景，使用C++性能会更高效

2. 实现成本问题

   对于有些功能，已有开源的C++的类库，但是没有JS库，而要重新实现相同功能的JS库难度太大，此时可以选用C++扩展



# 写C++扩展的开发环境准备

## 什么是node-gyp

node-gyp是Node.js下内置的C++扩展构建工具，它使用GYP工具通过一个名为 binding.gyp 的描述文件生成不同系统在构建和编译时所需要的C++项目文件（如Makefile）。

> GYP全称为Generate Your Project，是谷歌为Chromium写的构建工具



在每一个Node.js包中，如果需要编译C++代码，最通用的做法是把代码都放到某一个目录中，通过binding.gyp文件定义好这些文件、编译器指令等，在`npm install`执行时，node-gyp会被自动调用用于编译C++代码，默认是编译成一个动态链接库，并将动态链接库放入项目或包根目录的build目录下。

> 因为node-gyp是Node.js官方支持的，不需要嵌入任何脚本。如果使用其他构建工具，可以在package.json中的scripts字段加入postinstall的脚本，表示npm在安装完当前包时要执行的脚本，使其在包安装阶段执行构建



## 安装node-gyp

**全局安装node-gyp**

`npm install -g node-gyp`



**node-gyp的依赖项如下**

> [参考官方文档](https://github.com/nodejs/node-gyp)

| 平台    | 依赖                                                         |
| ------- | ------------------------------------------------------------ |
| unix    | 1、Python v3.7, v3.8, v3.9, or v3.10<br>2、make构建工具<br>3、C++编译工具（如GCC） |
| macOS   | 1、Python v3.7, v3.8, v3.9, or v3.10<br>2、Xcode开发工具，Xcode包含了各种构建需要的工具，安装完Xcode后安装Command Line Tools命令行工具（包含了clang和make） |
| Windows | 1、安装Visual C++构建环境：Visual C++ build tools 或 Desktop development with C++<br>2、npm config set msvs_version 2017 |



## node-gyp命令

**为指定版本的Node.js安装开发环境的文件**

```bash
node-gyp install
```

install命令会将当前执行的Node.js源码的一些头文件、lib文件等下载到当前用户的.node-gyp目录下，如/Users/sinkhaha/.node-gyp目录下，以Node.js相应的版本号为目录，如10.13.0版本的目录结构如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221117221430.png)



**生成配置文件**

```bash
# 为当前模块生成配置文件（如windows下是MSVC，unix是Makefile）
node-gyp configure
```

在构建源码前，必须先运行configure命令生成项目文件，运行后会在当前目录生成build目录，在unix环境下该目录是Makefile文件和一些必要的配置文件



**构建模块**

```bash
# 调用构建工具构建模块（unix是make、windows是msbuild）
node-gyp build
```

build会将当前目录的模块进行构建，将C++代码编译成二进制文件，运行后会在build/Release目录下生成一个xxx.node的二进制文件，此文件就是Node.js的一个C++模块，xx是binding.gyp里配置的编译后的文件名字



**清空构建产物**

```bash
# 清空构建时生成的构建文件以及out目录
node-gyp clean
```



**重新构建(一步到位)**

> 一般构建使用该命令就可以

```bash
# 一次性依次执行 clean、configure、build这3个命令
node-gyp rebuild
```



**列出安装的Node.js的头文件**

```bash
# 输出当前安装的不同版本的Node.js开发环境的头文件
node-gyp list

输出如下
> node-gyp list
gyp info it worked if it ends with ok
gyp info using node-gyp@9.3.0
gyp info using node@16.18.0 | darwin | x64
14.15.5
16.18.0
8.17.0
gyp info ok
```



**移除指定版本的Node.js头文件**

```bash
# 移除指定版本的Node.js开发环境文件
node-gyp remove <版本号>

如 node-gyp remove 8.17.0
```



**其他**

```bash
# 查看当前node-gyp的版本
node-gyp --version

# 查看帮助文档
node-gyp -h
```



# 实现C++扩展3种的方式

> 以下例子都基于Node.js  v16.18.0版本

## 1. Node原生

原生方式即直接使用Node.js底层的API和Chrome V8的API编写C++扩展。

> 16.x版本的Node.js内的[v8文档]( https://v8docs.nodesource.com/node-16.15/)



该方式的缺点：不同的Node.js版本底层用的V8版本不同（V8的API会不同），如果在代码中依赖了那些变化了的API，代码会直接编译不通过，导致代码不能使用。



### 实现hello-world例子

> [例子的github地址 ](https://github.com/sinkhaha/node-addon-demo/tree/main/hello-world-demo/node-v8)

**1. 编写binding.gyp文件**

> gyp文件是一个可配置、可编程的类JSON文件，可以写注释

```json
{
    "targets": [
        {
            "target_name": "hello", # 表示编译的结果是 hello.node
            "sources": [ # 表示要编译的源码
                "hello.cc"
            ]
        }
    ]
}
```

**2. 编写hello.cc源码**

```cc
#include <node.h>

namespace __v8_hello__{
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

// Hello方法的实现，该方法返回hello-world
void Hello(const FunctionCallbackInfo<Value>& args) {
    Isolate* isolate = args.GetIsolate();
    // Set方法用于设置返回值
    args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello-world").ToLocalChecked());
}

void init(Local<Object> exports) {
    // 在exports对象挂上hello属性，其值为Hello函数
    // NODE_SET_METHOD是一个宏，用于设置方法模板，它是node.h提供
    NODE_SET_METHOD(exports, "hello", Hello);
}

NODE_MODULE(addon, init)
}
```

**3. 测试文件test.js**

```js
// 导入编译好的hello.node二进制文件
const test = require('./build/Release/hello');

console.log(test.hello()); // 输出hello
```

**4. 编译并测试**

```bash
node-gyp install # 安装对应node版本的开发环境，已经安装过则跳过该命令

node-gyp rebuild # 即依次执行clean/configure/build命令 生成编译文件并构建

node test.js # 测试
```



## 2. NAN

NAN（Native Abstractions for Node.js）是Node.js原生模块抽象接口集。NAN是对各个版本的Node.js和V8的API的一个封装。



**NAN的本质**

NAN其实是一堆宏判断，NAN提供了很多宏，这些宏会判断当前编译的Node.js版本，根据不同版本的Node.js版本展开后执行不同的逻辑。



**NAN的特点**

一次编写，到处编译

> 只要Node.js的版本发生变化，NAN也得升级为已经兼容新Node.js的版本，用户的代码也跟着升级NAN版本，此时用户在新的Node.js版本下就可以被编译使用了。即一次编写好的代码，在不同的Node.js下重新编译



### 实现hello-world例子

> [例子的github地址]( https://github.com/sinkhaha/node-addon-demo/tree/main/hello-world-demo/nan)

**1. 安装nan**

```bash
npm i --save nan
```

**2. 编写binding.gyp文件**

```json
{
  "targets": [
    {
      "target_name": "hello",
      "sources": [ "hello.cc" ],
      "include_dirs": [ # 头文件的搜索路径，这样就能直接在项目中通过 #include <nan.h>来引入NAN
        "<!(node -e \"require('nan')\")" 
      ]
    }
  ]
}
```

**3. hello.cc源码**

```c++
#include <nan.h>

namespace __hello_nan__ {

using v8::String;
using v8::FunctionTemplate;

// Hello方法的实现，该方法返回hello-world
NAN_METHOD(Hello) {
    info.GetReturnValue().Set(Nan::New("hello-world").ToLocalChecked());
}
   
// Init方法的实现
NAN_MODULE_INIT(Init) {
    // target相当于module.exports对象
    // 在module.exports对象挂上hello属性，其值为Hello函数，也可以用以下的Nan::Export宏实现
    Nan::Set(target, Nan::New<String>("hello").ToLocalChecked(), Nan::GetFunction(Nan::New<FunctionTemplate>(Hello)).ToLocalChecked());
        
    // 也可以用“模块函数导出宏”
    // Nan::Export(target, "hello", Hello);
}

NODE_MODULE(hello, Init)
}
```

**4. test.js测试文件**

```js
let addon = require('./build/Release/hello'); // 直接导入

// 也可以用第三方的bindings包导入，bindings包用于用于加载c++模块
// let addon = require('bindings')('hello');

console.log(addon.hello()); // 输出 hello-world

```

**5. 编译并测试**

```bash
# 清空旧的构建产物、构建并编译
node-gyp rebuild

node test.js
```



 ## 3. N-API

从Node.js v8.0.0之后，Node.js推出了用于开发C++原生模块的接口N-API，它把Node.js的底层数据结构抽象成N-API的接口，不同Node.js版本使用同样的接口，这些接口是稳定的、ABI化的，即应用二进制接口（Application Binary Interface）。



**N-API的特点**

一次编写，到处编译

> 不同的Node.js版本，只要ABI的版本号一致，编译好的C++扩展就可以直接使用，不用重新编译(仅限同一系统)。
>
> 每个Node.js版本都会指定当前所使用的ABI版本，使用process.versions.napi查看Node.js的ABI版本



### 方式一：原生的Node-API 

Node-API是C语言风格的API，写起来会比较繁琐，社区为了方便，维护了一个C++风格的`node-addon-api`包。



#### 实现hello-world例子

> [例子的github地址](https://github.com/sinkhaha/node-addon-demo/tree/main/hello-world-demo/napi/Node-API) 

**1. 编写binding.gyp配置**

```json
{
  "targets": [
    {
      "target_name": "hello",
      "sources": [ "hello.c" ]
    }
  ]
}
```

**2. 编写hello.cc源码**

```c++
#define NAPI_VERSION 8
#include <assert.h>
#include <node_api.h>

// Hello方法的实现，该方法返回hello
static napi_value Hello(napi_env env, napi_callback_info info) {
  napi_status status;
  napi_value world;

  status = napi_create_string_utf8(env, "hello-world", 12, &world);

  assert(status == napi_ok);

  return world;
}


#define DECLARE_NAPI_METHOD(name, func) \
  { name, 0, func, 0, 0, 0, napi_default, 0 }

static napi_value Init(napi_env env, napi_value exports) {
  napi_status status;

  napi_property_descriptor desc = DECLARE_NAPI_METHOD("hello", Hello); // 设置exports对象的描述结构体

  status = napi_define_properties(env, exports, 1, &desc); // env表示底层上下文的参数

  assert(status == napi_ok);

  return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init) // 注册模块

```

**3. test.js测试文件**

```js
let addon = require('bindings')('hello');

console.log(addon.hello()); // 'hello world'
```

**4. 编译并测试**

```bash
npm i --save bindings # 用于加载c++模块

node-gyp rebuild

node test.js
```



### 方式二：第三方node-addon-api (推荐)

`node-addon-api`包是一个C++风格的包，也是由Node.js社区维护的，稳定性有一定的保证，写起来的代码比起Node-API会比较简洁。



#### 实现hello-world例子

>  [例子的github地址](https://github.com/sinkhaha/node-addon-demo/tree/main/hello-world-demo/napi/Node-API)

**1. 编写binding.gyp配置文件**

```json
{
  "targets": [
    {
      "target_name": "hello",
      "cflags!": [ "-fno-exceptions" ], # -fno-exceptions 忽略掉编译过程中的一些报错
      "cflags_cc!": [ "-fno-exceptions" ],
      "sources": [ "hello.cc" ],
      "include_dirs": [ # 头文件搜索路径，这样通过#include <napi.h>就可以引入napi
        "<!@(node -p \"require('node-addon-api').include\")"
      ],
      'defines': [ 'NAPI_DISABLE_CPP_EXCEPTIONS' ],
    }
  ]
}
```

**2. hello.cc源码**

```c++
#include <napi.h>

namespace __node_addon_api_hello__ {

// Hello方法的实现，该方法返回hello-world
Napi::String Hello(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();
  return Napi::String::New(env, "hello-world"); // 返回hello-world
}

Napi::Object Init(Napi::Env env, Napi::Object exports) {

  // 在exports对象挂上hello属性，其值为Hello函数，也可以用以下的Nan::Export宏实现
  exports.Set(Napi::String::New(env, "hello"),
              Napi::Function::New(env, Hello));

  return exports;
}

NODE_API_MODULE(addon, Init)

}
```

**3. test.js测试文件**

```js
// 用bindings包加载c++模块，也可以直接引入 let addon = require('./build/Release/hello');
let addon = require('bindings')('hello'); 

console.log(addon.hello()); // hello-world
```

**4. 编译并测试**

```bash
npm i --save node-addon-api
npm i --save bindings

node-gyp rebuild

node test.js
```

# 总结

本文讲解了什么是C++扩展，以及怎么使用，并为3种使用方式分别举了例子



# 参考

* [死月-《Node.js来一打C++扩展》]( https://book.douban.com/subject/30247892/)

* [GYP]( https://gyp.gsrc.io/)

* [Node.js库使用binding.gyp的例子]( https://github.com/nodejs/node-gyp/blob/main/docs/binding.gyp-files-in-the-wild.md)

* [node-addon-examples]( https://github.com/nodejs/node-addon-examples)

  
