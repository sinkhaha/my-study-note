# 1 隔离实例 Isolate

`Isolate` 是 `V8` 的一个引擎实例，表示 `JavaScript` 执行环境的对象，每个实例内部都有自己的各种状态（如 `JavaScript` 堆内存、垃圾回收等）。



每个 `Isolate` 对象都是相互独立的，所以在一个实例生成的任何对象不能在另外一个实例中使用，但可以在同一进程中创建多个独立的 `Isolate` 。



在 `Node.js` 中，每个 `JavaScript` 模块都在一个独立的 `Isolate` 对象中被执行，这说明每个模块都有自己的 JavaScript 堆内存和函数调用栈。

> `Node.js` 提供了一些 `API` 支持在不同的 `Isolate` 对象之间共享数据，例如 `v8::SharedArrayBuffer` 和 `v8::ArrayBuffer::Transfer` 等



`Node.js` 能使用 `C++` 扩展，其底层本质就是使用了 `V8` 暴露出来的 `API`。在开发 `Node.js` 的 `C++` 扩展时，就已经处于 `V8` 的环境，可以直接获取 `Node.js` 环境所使用的 `V8` 实例，如

```c++
// 获取 V8 实例
Isolate* isolate = args.GetIsolate();
```



# 2 上下文  Context

`Context` 是 `JavaScript` 执行上下文的一个对象，它包含了当前 `JavaScript` 线程的所有信息（如全局变量、函数调用栈等）。



`Context` 在创建时需要指定属于哪个实例，它起到隔离的作用。`Node.js` 中的 `vm` 模块，本质就是利用 `V8` 的允许多个上下文的特性。



在同一个 `Isolate` 实例中，可以有多个 `Context` 上下文，每个上下文是相互独立的，所以可以在同一线程中创建多个独立的  `JavaScript` 执行上下文

```c++
// 获取当前isolate的上下文
Local<Context> context = isolate->GetCurrentContext();
```



在 `Node.js` 中，每个 `JavaScript` 模块在一个独立的 `Context` 对象中被执行，这说明每个模块都有自己的全局变量和函数调用栈

> `Node.js` 提供了一些 `API` 支持在不同的 `Context` 对象之间共享数据，例如 `v8::Persistent` 和 `v8::HandleScope` 



# 3 脚本 Script

`Script` 是 `JavaScript` 脚本的对象，`Script` 是由 `JavaScript` 代码编译好的可执行对象。`Script` 对象可以被多次执行，而不必重新编译 `JavaScript` 代码。



要执行 `JavaScript` 代码，必须先为其指定一个 `Context` 上下文；例如以下代码，先将 `JavaScript` 代码编译为一个 `Script`  对象，接着该对象可以在 `V8` 引擎中执行该代码

```C++
// 将 JavaScript 代码编译为一个 Script 对象，source 表示一个包含 JavaScript 代码的字符串
Local<Script> script = Script::Compile(context, source).ToLocalChecked();
```



`Node.js` 使用 `vm` 模块来编译和执行 `JavaScript` 代码。`vm` 模块提供了 `vm.Script` 类，用于创建和管理 `Script` 对象，如

```js
const { Script } = require('vm');

const code = 'console.log("Hello, world!");';
// 创建Script对象
const script = new Script(code);

// 执行Script对象
script.runInThisContext();
```



# 4 常用数据类型

## 4.1 JavaScript 和 V8 数据类型

`JavaScript` 数据类型和 `V8` 数据类型是一一对应的关系，如下

| JavaScript数据类型 | JavaScript数据类型含义                                       | V8中每种 JavaScript 数据类型对应的 C++ 类型 | V8数据类型含义           |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------- | ------------------------ |
| `Undefined`        | 表示未定义的值                                               | `v8::Undefined`                             | 表示`Undefined` 类型的值 |
| `Null`             | 表示空值                                                     | `v8::Null`                                  | 表示 `Null `类型的值     |
| `Boolean`          | 表示布尔值                                                   | `v8::Boolean`                               | 表示 `Boolean` 类型的值  |
| `Number`           | 表示数字，包括整数和浮点数                                   | `v8::Number`                                | 表示 `Number` 类型的值   |
| `String`           | 表示字符串                                                   | `v8::String`                                | 表示 `String `类型的值   |
| `Object`           | 表示对象，包括普通对象、数组、函数、正则表达式等             | `v8::Object`                                | 表示 `Object` 类型的值   |
| `Symbol`           | 表示符号，是一种独一无二的标识符                             | `v8::Symbol`                                | 表示 `Symbol` 类型的值   |
| `BigInt`           | 一种特殊的数字类型，在 `JavaScript` 中可以表示任意精度的整数。`BigInt` 类型的值可以使用 `n` 后缀表示 | `v8::BigInt`                                | 表示 `BigInt` 类型的值   |

注意：`V8` 的数据类型（如 `v8::String`）跟原生的 `C++` 数据类型是不一样的，`V8` 的数据类型中都有对数据元信息的一个引用，用于内存管理



## 4.2 Value 类

`v8::Value` 是所有 `JavaScript` 数据类型的一个总的基类，其他数据类型都是继承这个 `Value`，而 `Value` 继承自 `Data` ；`External` 和 `Object` 和 元数据类型 `Primitive` 也继承自 `Value`。`V8` 中 `Value` 类的继承关系图如下



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20230424235045.png)



**`Value` 类型定义了两类通用的常用的方法：**

1. `Is` 开头的方法，用于判断 `Value` 是哪种数据类型，如
   * `IsString()`：判断该值是否为字符串类型
   * `IsUndefined()`：判断该值是否为 `undefined`

2. `To` 开头的方法，用于把 `Value` 转成另外一个类型的句柄，如
   * `ToString()`：将 `JavaScript` 值转换为字符串类型
   * `ToNumber()`：将 `JavaScript` 值转换为数字类型



# 5 句柄

`V8` 中的句柄是指对 `堆内存` 中 `JavaScript` 数据对象的一个引用，用于管理 `JavaScript` 对象。当一个 `JavaScript` 数据对象不再被句柄引用，它会被垃圾回收



**句柄的类型**

* 本地句柄 `v8::Local` （常用）
* 持久句柄 `v8::Persistent`（常用）
* 永生句柄 `v8::Eternal`
* 待实本地句柄 `MaybeLocal`
* 其他句柄



**句柄的声明**

句柄存在的形式是 `C++` 的一个模板类，它需要根据不同的 `V8` 数据类型进行不同的声明。例如

* `v8::Local<v8::Number>`：本地 `JavaScript` 数值类型句柄
* `v8::Persistent<v8::String>`: 持久 `JavaScript` 字符串类型句柄



**句柄的操作符**

* 点操作符：用来访问句柄对象的方法，如 `str.IsEmpty()` 判断这个句柄是否是空句柄

* `->` 操作符：获取这个句柄对象所引用的 `JavaScript` 对象的实体指针 

  > 如 `str->Length()`， `->` 符号等价于 `String*`，`String*` 即这个句柄所引用的 `JavaScript` 对象实体指针，且 `String` 有一个 `length` 方法可以获取字符串实体的长度



## 5.1 本地句柄

本地句柄是用于在当前作用域内管理 `JavaScript` 对象的引用，本地句柄存在于栈中



**生命周期**

本地句柄的生命周期与当前作用域相同，当作用域结束时，本地句柄将被自动销毁，所以它的生命周期是由其所存在的句柄作用域 `（Handle Scope）` 决定，本地句柄在对应的 `析构函数` 被调用时被删除



**创建方式**

1. 用 `New` 静态函数创建本地句柄

   例如 通过 `isolate` 跟另一个本地句柄，`Local<T>::New(Isolate* isolate, Local<T> that)`，`T` 是 `V8` 的 `JavaScript` 数据类型 或者 通过 `isolate` 跟一个持久句柄，`Local<T>::New(Isolate* isolate, const PersistentBase<T> &that)`



2. 通过 `V8` 的 `JavaScript` 数据类的静态方法获得本地句柄

   例如 `Local<Number> Number::New(Isolate* isolate, double value)` 就是一个 `v8::Number` 类型的静态方法，传入 `isolate` 和 `C++` 的一个 `double` 类型数据，`V8` 就会创建一个 `JavaScript` 中的 `Number` 数据，然后返回一个执行内存中引用该数据的本地句柄



**本地句柄的一些方法**

* `Clear()` 清除方法，用于设置为空句柄 ，可以简单理解为赋值为 `Null`，如

  ```c++
  Local<Number> num = Number::New(isolate, 123);
  num.clear();
  ```

  当一个句柄是空句柄时，对它进行操作会导致程序崩溃，使用前注意进行非空判断

* `IsEmpty()`  用于判断是否为空

  ```c++
  Local<Number> num = Number::New(isolate, 123);
  
  if (num.IsEmpty()) {
  }
  ```

* `As / Cast` 函数，用于转换数据类型，As是成员函数，Cast是静态函数，如

  ```c++
  Local<Value> handle = xxxx;
  
  Local<Number> num = Local<Number>::Cast(handle);
  或
  Local<Number> num = handle.As<Number>();
  ```

  

## 5.2 持久句柄

持久句柄是用于在多个作用域中管理 `JavaScript` 对象的引用，持久句柄存在于堆中



**生命周期**

持久句柄生命周期的管理方式跟本地句柄的不同。在当前作用域结束，持久句柄不会自动销毁，持久句柄则需要手动释放，以避免内存泄漏和性能瓶颈等问题。因此持久句柄使得一个 `JavaScript` 对象不只存在于当前的句柄作用域中，逃离了句柄作用域的管控



**持久句柄的类型**

* 唯一持久句柄 `UniquePersistent`

  > 用 `v8::UniquePersistent<...>` 表示，唯一持久句柄使用 `C++` 的 构造函数 和 析构函数 来管理其底层对象的生命周期 

* 一般持久句柄 `Persistent`（常用）

  > 用 `v8::Persistent<...>` 表示，一般持久句柄可以用它的构造函数创建，但是必须调用 `Persistent::Reset` 显示的清除，因为它不受控于句柄作用域的管控

* 弱持久句柄

  > 通过 `PersistentBase::SetWeak()` 方法使一个持久句柄变成弱持久句柄。



注意：当一个 `JavaScript` 对象的引用只剩一个弱持久句柄时，`V8` 的垃圾回收器就会触发一个回调（这个回调就是 `SetWeak()` 的参数传入的一个回调），Node.js 中会用此方式管理 `JavaScript` 对象跟 `C++` 对象的内存



**创建一般持久句柄**

一般持久句柄可以通常通过本地句柄升格而成，得到一个本地句柄所引用的 `V8` 数据对象的一个持久句柄

* 直接用 `Persistent()`创建，后面通常再调用别的方法对一个本地句柄升格
* 用 `Persistent(Isolate *isolate, Local<T> that)`：传入一个 `Isolate` 和一个本地句柄



**持久句柄的其他方法**

* `SetWeak()`：用于设置为弱持久句柄（常用）

* `ClearWeak()`：取消它的弱持久句柄，又变成一般持久句柄



## 5.3 待实本地句柄

通过 `MaybeLocal` 类实现，它表示待落实它是不是一个有效的本地句柄。



本地句柄如果用 `Clear()` 设置为空句柄，每次在使用本地句柄时都要注意进行非空判断，这会带来一定的心智负担，让我们忘记进行判断导致出错。使用 `Maybe Local` 就可以提醒这是一个待落实的本地句柄，让用户知道这个地方的返回值需要检查是否为空，如果要拿到真正的本地句柄，待落实本地句柄可以调用它的 `ToLocalChecked` 函数，例如

```C++
MaybeLocal<String> s = x.ToString();
Local<String> _s = s.ToLocalChecked();
```



## 5.4 永生句柄

永生句柄通过 `v8::Eternal` 类实现，它是一种特殊的持久句柄，用于在整个程序运行期间管理 `JavaScript` 对象的引用。



与普通的持久句柄不同，永生句柄不会在内存中被释放，因此可以保证被引用的 `JavaScript` 对象不会被 `V8` 引擎的垃圾回收机制回收，所以永生句柄在程序的整个生命周期都不会被删除。



# 6 句柄作用域

句柄作用域是用于管理 `本地句柄` 的生命周期。句柄作用域可以确保本地句柄不会被垃圾回收器回收，从而避免出现野指针和内存泄漏等问题。



句柄作用域也是存在于栈内存中，作用域是一个套一个以栈的形式存在的。当创建一个新的句柄作用域时，该句柄作用域会被添加到当前的句柄作用域栈中。当句柄作用域被销毁时，该句柄作用域中的所有本地句柄都会被自动释放。



句柄作用域实际是维护一堆本地句柄的容器。当一个句柄作用域对象的 `析构函数` 被调用时，在这个句柄作用域中创建的所有本地句柄都会从栈中删除，通常这些本地句柄所指的对象将会失去该引用，如果它们没有别的地方有引用的话，就可以被下一次垃圾回收进行回收了。



**两种句柄作用域类型**

* 一般句柄作用域 `HandleScope`

  一个 `HandleScope` 对象会在一个函数内的一开始被声明；在函数结束时被删除，此时会自动调用它的解构函数做一些事情

* 可逃逸句柄作用域 `EscapableHandleScope`

  `EscapableHandleScope`  是一种特殊的句柄作用域类型，用于在句柄作用域内创建本地句柄，并将其传递到句柄作用域之外。可以使用 `EscapableHandleScope` 的 `Escape` 函数，把一个本地句柄复制到一个封闭的作用域中，并删除其他的本地句柄，然后返回这个新复制的本地句柄，即得到一个可以被安全返回的句柄



**使用 一般句柄作用域 和 可逃逸句柄作用域的例子**

有以下 `JavaScript` 代码：

```js
function returnValue() {
    let number = 2333;
    return number
}
```

接着使用 `C++` 实现上面等价的 `JavaScript` 代码。

分析：因为有返回值 `number`，所以要使用可逃逸句柄作用域把本地句柄返回给外部；如果只是使用一般句柄作用域，那这个一般句柄作用域在函数结束时被删除，其管理的本地句柄也被删除，此时不能返回给外部。



使用一般句柄作用域的例子：

```c++
v8::Local<v8::Number> ReturnValue() {
    v8::Isolate* isolate = v8::Isolate::GetCurrent();
    
    // 一般句柄作用域
    v8::HandleScope scope(isolate);
  
    v8::Local<v8::Number> number = v8::Number::New(Isolate, 2333);
  
    // 会被垃圾回收，不能返回
    return number;
}
```

这个例子会有问题，需要将上面一般句柄作用域改成使用可逃逸句柄作用域，返回一个安全的句柄



使用可逃逸句柄作用域的例子：

```C++
v8::Local<v8::Number> ReturnValue() {
    v8::Isolate* isolate = v8::Isolate::GetCurrent();
    
    // 可逃逸句柄作用域
    v8::EscapeHandleScope scope(isolate);
  
    v8::Local<v8::Number> number = v8::Number::New(Isolate, 2333);
  
    // 使用Escape方法
    return scope.Escape(number);
}
```



# 7 模板

`V8`  的模板是指在上下文中 `JavaScript` 对象以及函数的一个模具。可以用一个模板把自己 `C++` 函数 或 数据结构 包裹进 `JavaScript` 的对象中，这样就可以使用 `JavaScript` 代码来对它们进行一些操作了。

> `C++` 扩展实际上也就是让 `C++` 代码能被 `Node.js` 中的 `JavaScript` 代码所调用



## 7.1 函数模板

函数模板在 `V8` 的数据类型是 `FunctionTemplate` ，`FunctionTemplate` 继承自 `Template`，它是一个用于创建 `JavaScript` 函数的模具。



**创建函数模板**

可以通过 `FunctionTemplate::New(xxx)` 来创建一个函数模板，如

```C++
// 实例化函数模板
Local<FunctionTemplate> ft = FunctionTemplate::New(isolate, Method); // 将Method这个函数包裹
```



**实例化**

`GetFunction()` 用于实例化一个 `JavaScript` 可用的函数。函数模板的各种定义，最终就是为了能实例化成一个 `JavaScript` 中可用的函数；创建一个函数模板后，并调用它的 `GetFunction` 方法获得其函数实体句柄后，这个函数就能被 `JavaScript` 所调用，如

```c++
// 实例化函数模板
Local<FunctionTemplate> ft = FunctionTemplate::New(isolate, Method); // 将Method这个函数包裹，生成一个函数模板
Local<Function> fn = ft->GetFunction(context).ToLocalChecked(); // 得到一个js中能用的函数实例Function     
```



**设置本体**

`SetCallHandler` 用于设置函数模板的函数本体。因为在用 `New` 创建一个函数模板时，`callback` 是可以不传的，如果不传，此时生成的函数模板是不带函数本体的



**设置类名**

`SetClassName` 用于设置类名，当一个函数模板实例化出来的函数若作为一个构造函数的话，则 `SetClassName` 设置的类名就是我们当通过 `JavaScript` 的 `new` 操作符实例化出来的对象时对应的类名



**设置参数数量**

`SetLength` 用于设置函数在定义时的参数数量



**使用函数模板的例子**

> 完整代码可见[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/v8-api/function-template)

这是一个 `node addon` 的例子，里面使用了 `V8` 的 `API`， 注意不同 `Node.js` 版本 `V8` 的 `API` 有差异，运行时要根据具体的 `Node.js` 版本调整 `V8` 的 `API`，不同的 `Node.js` 版本使用的 `V8 API` 可见[此文档](https://v8docs.nodesource.com/)

```c++
#include <node.h>

// 函数模板
namespace __functionTemplate__ {
 
/**
 * Method方法有单一参数FunctionCallbackInfo<Value>的引用，返回值是void类型
 */
void Method(const FunctionCallbackInfo<Value>& args) {
    Isolate* isolate = args.GetIsolate();

    // 设置方法返回值为 this is function
    args.GetReturnValue().Set(String::NewFromUtf8(isolate, "this is function").ToLocalChecked());
}

void init(Local<Object> exports) {
    Isolate* isolate = Isolate::GetCurrent(); // 获取v8实例
    HandleScope scope(isolate); // 实例化句柄对象

    Local<Context> context = isolate->GetCurrentContext(); // 获取上下文

    // 函数模板的用法，设置函数名
    // 实例化函数模板
    Local<FunctionTemplate> ft1 = FunctionTemplate::New(isolate, Method); // 将Method这个函数包裹，生成一个函数模板
    Local<Function> fn1 = ft1->GetFunction(context).ToLocalChecked(); // 得到一个js中能用的函数实例Function
       
    // 设置函数名为testCreateByTemplate
    Local<String> fnName1 = String::NewFromUtf8(isolate, "testCreateByTemplate").ToLocalChecked();
    fn1->SetName(fnName1);
       
    // 挂载testCreateByTemplate函数到exports对象
    exports->Set(context, fnName1, fn1).Check();

}

NODE_MODULE(addon, init)

}
```

这里 `Method` 的参数是`FunctionCallbackInfo` 类型，在 `V8` 中，`FunctionCallbackInfo` 是一个用于传递函数调用信息的类。当我们在 `C++` 中编写 `V8` 中的函数回调时，需要使用 `FunctionCallbackInfo` 类来获取函数参数和返回值等信息。



## 7.2 对象模板

对象模板类型在 `V8` 的数据类型是 `ObjectTemplate`，`ObjectTemplate` 也是继承自 `Template`，用于创建对象。



**创建对象模板**

可以通过 `ObjectTemplate::New(xxx)` 来创建一个对象模板

```C++
Local<ObjectTemplate> obj_with_func_tpl = ObjectTemplate::New(isolate);
```



**设置函数体**

`SetCallAsFunctionHandler` 用于设置函数体，就是配置一下这个函数模板。例如有以下 JavaScript 代码 

```js
function func() {
    return 2333;
}

func.cat = '小猫';
func.dog = '小狗';

console.log(func);
console.log(func());
console.log(func.cat);
console.log(func.dog);
```

接下来，用 `C++` 代码实现以上功能，此时可以使用 `SetCallAsFunctionHandler` ，如

```c++
void Func(const FunctionCallbackInfo<Value>& args) {
  args.GetReturnValue().Set(2333);
}

Local<ObjectTemplate> obj_with_func_tpl = ObjectTemplate::New(isolate);
// 对象设置cat属性，值为小猫
obj_with_func_tpl->Set(String::NewFromUtf8(isolate, "cat").ToLocalChecked(), String::NewFromUtf8(isolate, "小猫").ToLocalChecked());

// 对象设置dog属性，值为小狗
obj_with_func_tpl->Set(String::NewFromUtf8(isolate, "dog").ToLocalChecked(), String::NewFromUtf8(isolate, "小狗").ToLocalChecked());

obj_with_func_tpl->SetCallAsFunctionHandler(Func); // 设置函数体为Func方法

Local<Object> newInstance = obj_with_func_tpl->NewInstance(context).ToLocalChecked();

// 暴露func方法
exports->Set(context, String::NewFromUtf8(isolate, "func").ToLocalChecked(), newInstance);
```



**对象模板的使用方式**

> 完整代码可见[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/v8-api/object-template)

* 每个函数模板都有一个与之关联的对象模板

> 当函数模板被用作一个构造函数时，这个对象模板就是用来配置创建出来的对象，这个对象模板就是原型链模板（`PrototypeTemplate`），原型链模板实际上是一个对象模板，只不过在函数模板中作为原型链使用，如

```c++
// 用法1: 函数模板有一个与之关联的对象模板(此时是原型链对象模板)

// 新建一个函数模板，并设置类名为TestClass，其函数体为Constructor函数
Local<FunctionTemplate> ft = FunctionTemplate::New(isolate, Constructor); 
Local<String> className = String::NewFromUtf8(isolate, "TestClass").ToLocalChecked();
ft->SetClassName(className);

// 获取函数模板的原型对象，并设置为get函数，其函数体为ClassGet函数
Local<ObjectTemplate> proto = ft->PrototypeTemplate();
proto->Set(String::NewFromUtf8(isolate, "get").ToLocalChecked(), FunctionTemplate::New(isolate, ClassGet));

// 暴露TestClass对象，其值为TestClass函数
exports->Set(context, className, ft->GetFunction(context).ToLocalChecked()).Check();;
```

* 对象模板也能脱离函数模板使用，如

```c++
// 用法2：对象模板脱离函数模板使用，作为对象模板创建对象

//新建一个函数模板，并设置类名为TestClass2
Local<FunctionTemplate> fun = FunctionTemplate::New(isolate);
fun->SetClassName(String::NewFromUtf8(isolate, "TestClass2").ToLocalChecked());

// 新建一个对象模板
Local<ObjectTemplate> obj_tpl = ObjectTemplate::New(isolate, fun);
// 对象模板设置一个num属性，值为233
obj_tpl->Set(String::NewFromUtf8(isolate, "num").ToLocalChecked(), Number::New(isolate, 233)); 
    
// 实例一个3个长度的数组，往里面存入对象
Local<Array> array = Array::New(isolate, 3);
for(int i = 0; i < 3; i++) {
    array->Set(context, Number::New(isolate, i), obj_tpl->NewInstance(context).ToLocalChecked()).Check(); // NewInstance()每个实体不同
}

// 把整个数组挂到exports的array属性
exports->Set(context, String::NewFromUtf8(isolate, "array").ToLocalChecked(), array).Check();;
```



## 7.3 对象模板的内置字段

因为 `V8` 中能与 `JavaScript` 代码直接交互的数据类型都是以句柄形式出现的 `V8` 数据类型，如 `v8::String` 、`v8::Number` 等。那我们自定义的 `C++` 数据结构怎样才能被 `JavaScript` 代码使用？此时就需要用到内置字段。



`Internal Field` 内置字段可以把一个我们自己的 `C++` 数据结构对象设置为 `V8` 数据对象的一个内置字段，使得我们自己的 `C++` 数据结构 和 `V8` 的数据类型对象相关联，这样 `JavaScript` 代码就能用到该数据结构。



注意：内置字段可以理解为 `V8` 对象数据类型的私有属性，对于 `JavaScript` 代码是不可见的，只有在 `V8` 的 `C++` 层面才能通过 `v8::Object` 的特定方法将其取出来



可以使用 `SetInternalField` 和 `SetAlignedPointerInInternalField` 设置内置字段。



**SetInternalField 的使用**

> 完整代码可见[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/v8-api/internal-field)

```c++
#include <node.h>

namespace __internal_field_right__ {

// 自定义数据结构，内部加了一个persistent持久句柄，作用是用于回收自定义字段
struct PersistentWrapper {
    Persistent<Object> persistent; // 持久句柄
    int value;
};

/**
 * 弱持久句柄的回调函数
 * 当对象被回收时，此回调会被触发
 */
void WeakCallback(const v8::WeakCallbackInfo<PersistentWrapper>& data)
{
    PersistentWrapper* wrapper = data.GetParameter(); // 获取到SetWeak方法传的第一个参数，此时是wrapper参数

    printf("deleting 0x%.8X: %d...", wrapper, wrapper->value);

    wrapper->persistent.Reset(); // 重置(释放)这个持久句柄

    delete wrapper; // 删除wrapper对象，即删除堆内存分配的指针

    printf("ok\n");
}

/**
 * 当obj对象被垃圾回收时，会触发WeakCallback回调，这个回调里主要是删除new出来的PersistentWrapper对象
 */
void CreateObject(const FunctionCallbackInfo<Value>& args)
{
    Isolate* isolate = args.GetIsolate();
    Local<Context> context = isolate->GetCurrentContext();

    Local<ObjectTemplate> templ = ObjectTemplate::New(isolate);
    // 设置对象模板的内置字段数为1
    templ->SetInternalFieldCount(1);

    // 实例子化PersistentWrapper
    PersistentWrapper* wrapper = new PersistentWrapper();
    // 方法传入的第一个参数赋值给value
    wrapper->value = args[0]->ToNumber(context).ToLocalChecked()->Int32Value(context).FromJust();
    
    // obj模板对象设置内置字段，值为wrapper对象
    Local<Object> obj = templ->NewInstance(context).ToLocalChecked();
    obj->SetInternalField(0, External::New(isolate, wrapper)); // 时内置字段是一个 External 永生句柄类型

    // 重置(释放)wrapper的持久句柄
    wrapper->persistent.Reset(isolate, obj);
    // 将持久句柄设置为弱持久句柄，回调函数为WeakCallback（当被回收时被调用）。wrapper参数可以在WeakCallback方法中获取到
    wrapper->persistent.SetWeak(wrapper, WeakCallback, v8::WeakCallbackType::kInternalFields);

    args.GetReturnValue().Set(obj);
}

void Init(Local<Object> exports, Local<Object> module)
{
    Isolate* isolate = Isolate::GetCurrent();
    Local<Context> context = isolate->GetCurrentContext();
    HandleScope scope(isolate);

    // 暴露create函数，其函数体为CreateObject
    exports->Set(context, String::NewFromUtf8(isolate, "create").ToLocalChecked(), FunctionTemplate::New(isolate, CreateObject)->GetFunction(context).ToLocalChecked()).Check();
}

NODE_MODULE(_template, Init)

}

```

分析

* 在设置内置字段时，要使用 `SetInternalFieldCount()` 设置对象模板的内置字段数

* 此时内置字段是一个 `External` 永生句柄类型，`External` 对象里的内置字段不属于 `V8` 的管辖范畴，不会被 `V8` 垃圾回收，需要我们自己掌控这些内置字段的生命周期（一个内置字段如果回收早了，可能下次访问它会导致奔溃，如果回收晚了，则可能会内存泄漏）

* 当这个函数返回的对象 `obj` 被 `V8` 垃圾回收时，就会触发一个 `WeakCallback` 回调，在这个回调中删除在这里 `new` 出来的 `PersistentWrapper` 

  > `Node.js` 中的 `ObjectWrapper` 类也是利用了内置字段




**内置字段的回收时机**

因为 `SetInternalField` 会把内置字段设置为 `External` 类型，通常内置字段本体必须与它的宿体 `External` 数据对象共存亡，即在 `External` 句柄被 `V8` 回收时做一些事件（比如用 `delete` 删除这个内置字段本体）



可以利用弱持久句柄回收内置字段：当一个 `JavaScript` 对象的引用只剩下一个弱持久句柄时，`V8` 的垃圾回收器就会触发一个回调（这个回调可以由我们传入），也就是弱持久句柄指向的 `V8` 数据体的所有其他类型的句柄引用全部都没有了，它即将要被垃圾回收时，这个回调就会触发



**SetAlignedPointerInInternalField 的使用**

> 使用 SetAlignedPointerInInternalField 的例子，可见[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/v8-api/aligned-pointer-internal-field)的代码

`V8` 的 `SetAlignedPointerInInternalField` 可用于将指针存储在 `JavaScript` 对象的内部字段中，作用和 `SetInternalField` 一样。



**SetAlignedPointerInInternalField 和 SetInternalField()的区别**

* 都是用于在 `JavaScript` 对象的内部存储一些额外的数据

* `v8::SetInternalField` 可以将一个指针 或 整数值存储在 JavaScript 对象的内部字段中，它可以存储除指针外的其他类型的数据，如 `JavaScript` 对象或字符串；它会将 `value` 存储在对象的第 `index` 个内部字段中，并将字段的对齐方式设置为值的对齐方式

* `v8::SetAlignedPointerInInternalField` 方法只能用于存储指针类型的数据，并将字段的对齐方式设置为指针的对齐方式。这样可以保证对齐的正确性，提高访问效率，同时也可以方便地在 `C++` 代码中访问这些指针类型的数据



## 7.4 对象模板的继承

`V8` 的函数模板提供了一个函数 `Inherit()`，可以将一个函数模板继承自另一个函数模板



**使用继承例子**

> 完整代码可见[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/v8-api/function-template-inherit)

```c++
#include <node.h>

namespace __inherit__ {

Persistent<Function> cons;

void SetName(const FunctionCallbackInfo<Value>& args)
{
    Isolate* isolate = args.GetIsolate();
    Local<Context> context = isolate->GetCurrentContext();

    Local<Object> self = args.Holder();

    // 等价于 "this.name = 传进来的第一个参数"
    self->Set(context, String::NewFromUtf8(isolate, "name").ToLocalChecked(), args[0]).Check();
}

void Summary(const FunctionCallbackInfo<Value>& args)
{
    Isolate* isolate = args.GetIsolate();
    Local<Context> context = isolate->GetCurrentContext();

    Local<Object> self = args.Holder();
    char temp[512];

    String::Utf8Value type(isolate, self->Get(context, String::NewFromUtf8(isolate, "type").ToLocalChecked()).ToLocalChecked()->ToString(context).ToLocalChecked());
    String::Utf8Value name(isolate, self->Get(context, String::NewFromUtf8(isolate, "name").ToLocalChecked()).ToLocalChecked()->ToString(context).ToLocalChecked());

    // 把值赋给temp 等价于js的`${this.name} is a/an ${this.type}`
    snprintf(temp, 511, "%s is a/an %s.", *name, *type);

    // 返回temp
    args.GetReturnValue().Set(String::NewFromUtf8(isolate, temp).ToLocalChecked());
}

void Pet(const FunctionCallbackInfo<Value>& args)
{
    Isolate* isolate = args.GetIsolate();
    Local<Context> context = isolate->GetCurrentContext();

    Local<Object> self = args.Holder();

    // 等价于 js 代码
    // this.name = 'Unknown';
    // this.type = 'animal';
    self->Set(context, String::NewFromUtf8(isolate, "name").ToLocalChecked(), String::NewFromUtf8(isolate, "Unknown").ToLocalChecked()).Check();
    self->Set(context, String::NewFromUtf8(isolate, "type").ToLocalChecked(), String::NewFromUtf8(isolate, "animal").ToLocalChecked()).Check();

    // 返回this
    args.GetReturnValue().Set(self);
}

void Dog(const FunctionCallbackInfo<Value>& args)
{
    Isolate* isolate = args.GetIsolate();
    Local<Context> context = isolate->GetCurrentContext();

    Local<Object> self = args.Holder(); // 获取this对象
    Local<Function> super = cons.Get(isolate); // 获取super，即Pet

    // 即等价于js代码的 Pet.call(this)
    super->Call(context, self, 0, NULL).ToLocalChecked();

    // 等价于js代码 this.type = 'dog'
    self->Set(context, String::NewFromUtf8(isolate, "type").ToLocalChecked(), String::NewFromUtf8(isolate, "dog").ToLocalChecked()).Check();

    // 返回自身 this
    args.GetReturnValue().Set(self);
}

void init(Local<Object> exports, Local<Object> module)
{
    Isolate* isolate = Isolate::GetCurrent();
    Local<Context> context = isolate->GetCurrentContext();
    HandleScope scope(isolate);

    // 函数模板Pet
    Local<FunctionTemplate> pet = FunctionTemplate::New(isolate, Pet);
    // PrototypeTemplate返回了这个函数模板的原型链上的原型对象模板prototype
    // 在Pet的原型对象上设置setName函数和summary函数
    pet->PrototypeTemplate()->Set(
            String::NewFromUtf8(isolate, "setName").ToLocalChecked(),
            FunctionTemplate::New(isolate, SetName));
    pet->PrototypeTemplate()->Set(
            String::NewFromUtf8(isolate, "summary").ToLocalChecked(),
            FunctionTemplate::New(isolate, Summary));

    Local<Function> pet_cons = pet->GetFunction(context).ToLocalChecked();

    cons.Reset(isolate, pet_cons);

    // 函数模板Dog
    Local<FunctionTemplate> dog = FunctionTemplate::New(isolate, Dog);
    // Dog继承Pet
    dog->Inherit(pet); 

    Local<Function> dog_cons = dog->GetFunction(context).ToLocalChecked();
    
    // 暴露Pet和Dog函数
    exports->Set(context, String::NewFromUtf8(isolate, "Pet").ToLocalChecked(), pet_cons).Check();
    exports->Set(context, String::NewFromUtf8(isolate, "Dog").ToLocalChecked(), dog_cons).Check();
}

NODE_MODULE(addon, init)

}
```



## 7.5 对象模板的访问器与拦截器

* 访问器：对象指定属性被访问时执行，使用 `SetAccessor`

* 拦截器：对象任何属性被访问时执行，使用 `SetHandler`



拦截器又分为

* 映射型拦截器：一个对象内成员的访问方式是 `字符串型的属性名` 时，该拦截器会生效
* 索引型拦截器：与数组类似，当通过整型下标对内容进行访问时，该拦截器会生效



# 8 总结

本文主要介绍了 `V8` 的基础概念，如实例、上下文、脚本、常用的数据类型、句柄、句柄作用域、模板等。并通过 `C++` 扩展等方式演示了如何使用这些对象



# 9 参考

* [Getting started with embedding V8](https://v8.dev/docs/embed)
* [V8 of Node.js](https://v8docs.nodesource.com/)
* [Node.js来一打C++扩展](https://book.douban.com/subject/30247892/)

