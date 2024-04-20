> 本文 `Node.js` 的版本` v18.15.0` 、`libuv` 的版本 `v1.44.2` （`libuv` 相关只涉及 `unix` 平台）
>
> 前置知识：需要先了解 `Node.js` 中 `JS` 层跟 `C++` 层的交互知识，可以参考[这篇文章](https://zhuanlan.zhihu.com/p/629195164)

# 1 BaseObject 类的 MakeWeak() 函数

在 `Node.js` 中，普通的 `JS` 对象当没有被引用时会被垃圾回收。但也有特殊的情况： `JS` 对象关联了 `C++` 对象的情况，这时我们要保证 `JS` 对象跟 `C++` 对象的共存亡；如果 `JS` 对象被回收了，但是 `C++` 对象没被回收，那会导致内存泄漏。针对这种情况，`Node.js` 的解决方式是利用了 `V8` 提供的弱持久句柄的回调机制：当一个 `JS` 对象的引用只剩一个弱持久句柄时，该 `JS` 对象就可以被回收了，当 `JS` 对象被回收后会执行句柄通过 `SetWeak() `函数设置的回调函数，在该回调函数中我们可以释放 `C++` 对象的内存。`Node.js` 中`BaseObject` 类的 `MakeWeak()` 函数就提供了这个弱持久句柄的回调机制，这是 `Node.js` 中大多 `C++` 对象垃圾回收的基础，下面先来看下 `MakeWeak()` 函数的实现。



**MakeWeak() 代码如下**

`BaseObject` 类是所有自己定义 `JS` 层对象的基类；其中的 `MakeWeak()` 函数用于管理 `JS` 层对象和 `C++` 对象的内存，本质是利用了 `V8` 弱持久句柄的机制。它的定义和实现如下

```c++
// MakeWeak的定义 
// node-18.15.0/src/base_object.h
void MakeWeak();

// 实现 
// node-18.15.0/src/base_object.cc
void BaseObject::MakeWeak() {
  if (has_pointer_data()) {
    pointer_data()->wants_weak_jsobj = true;
    if (pointer_data()->strong_ptr_count > 0) return;
  }

  // 将持久句柄设置为弱持久句柄，并注册一个回调，当js对象被垃圾回收时，会触发该回调，在回调中，释放C++对象占用的资源
  persistent_handle_.SetWeak(
      this,
      // 回调函数
      [](const WeakCallbackInfo<BaseObject>& data) {
        BaseObject* obj = data.GetParameter();
        // Clear the persistent handle so that ~BaseObject() doesn't attempt
        // to mess with internal fields, since the JS object may have
        // transitioned into an invalid state.
        // Refs: https://github.com/nodejs/node/issues/18897
        obj->persistent_handle_.Reset(); // 清除persistent_handle_持久句柄，即不再引用 JS 对象
        CHECK_IMPLIES(obj->has_pointer_data(),
                      obj->pointer_data()->strong_ptr_count == 0);
        obj->OnGCCollect(); // 释放 BaseObject 对象内存
      },
      WeakCallbackType::kParameter);
}

// base_object-inl.h
// 删除 BaseObject 对象
void BaseObject::OnGCCollect() {
  delete this;
}
```

重点：`BaseObject` 类的 `persistent_handle_` 句柄变量保存了一个 `JS` 层句柄对象；当 `BaseObject` 的子类调用 `MakeWeak()` 函数后，`persistent_handle_` 句柄会调用 `SetWeak()` 函数并设置一个回调函数（`SetWeak()` 的作用是当 `JS` 层对象只有 `persistent_handle_` 这一个引用时，这个 `JS` 层对象也会被垃圾回收，`V8` 在垃圾回收时就会执行 `SetWeak()` 传入的回调函数）。这里的回调函数会通过调用 `Reset()` 重置 `persistent_handle_` 持久句柄，即使得 `persistent_handle_` 句柄不再引用 `JS` 层对象，从而释放 `persistent_handle_` 对象的内存空间，最后再调用 `OnGCCollect()` 释放 `BaseObject` 对象的内存。



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/BaseObject%E7%BB%93%E6%9E%84%E5%9B%BE.png)

注意： `BaseObject` 默认不会调用 `MakeWeak()`，而是通过其子类调用

> 比如下面要讲的 `trace_events` 模块中的 `NodeCategorySet` 子类就会在其构造函数中调用 `MakeWeak()`



# 2 关联了 C++ 的 JS 层对象的内存管理

对于关联了 `C++` 对象的 `JS` 层对象，就需要通过 `BaseObject` 类的 `MakeWeak()` 函数设置弱引用回调，这样当 `JS` 对象失去其他引用而被回收时，就会触发 `MakeWeak()` 设置回调函数，在回调函数中 `JS` 层对象关联的 `C++` 对象也会被回收，否则就会造成内存泄露。`trace_events` 模块的内存管理就是利用了这个机制。下面以` trace_events` 模块为例了解下关联了 `C++` 对象的 `JS` 层对象的内存管理机制。



## 2.1 例1 

先看下面的例子了解下 `trace_events` 的使用

```js
// test.js
setInterval(() => {
    gc();
    console.log('手动触发gc');
}, 1000);

const trace_events = require('trace_events');
// 注意：这里没使用createTracing返回的对象，所以执行结束后，Tracing对象就会被垃圾回收，如果把 Tracing对象赋值给 global 全局对象，则 Tracing对象不会被垃圾回收
trace_events.createTracing({categories: ['node.perf', 'node.async_hooks']}); // categories为要跟踪的事件类别

// 运行 node --expose-gc test.js  指定--expose-gc才能手动触发gc
```

`createTracing()` 用于创建跟踪会话，可以生成跟踪事件，并将这些事件保存到文件中，以便后续分析和调试。



思考一下：这个例子执行完的内存回收流程是怎么的，会回收哪些内存？

> 要解答这个问题，需要先了解下 `trace_events` 调用 `createTracing` 后的内部原理。需要回收的内存如下
>
> * `JS` 层的 `Tracing` 对象的内存
> * `Tracing` 对象的成员变量-- `CategorySet` 对象的内存
> * `C++` 对象 `NodeCategorySet` 的内存



## 2.2 trace_events 模块分析

`trace_events` 模块的 `JS` 层代码如下

```js
// node-18.15.0/lib/trace_events.js
const { CategorySet } = internalBinding('trace_events'); // node_trace_events.cc

//  createTracing 可以创建一个收集 trace event 数据的对象
function createTracing(options) {
  validateObject(options, 'options');
  validateStringArray(options.categories, 'options.categories');

  if (options.categories.length <= 0)
    throw new ERR_TRACE_EVENTS_CATEGORY_REQUIRED();

  // 实例化Tracing，并返回，Tracing是js层自己实现的类
  return new Tracing(options.categories);
}

// node-18.15.0/lib/trace_events.js
class Tracing {
  constructor(categories) {
    // 实例化 CategorySet，并保存在kHandle变量中
    this[kHandle] = new CategorySet(categories);
    this[kCategories] = categories;
    this[kEnabled] = false;
  }
  ... 
}
```

由代码可知，`createTracing()` 中实例化了一个 `Tracing` 对象，`Tracing` 对象里保存了 `CategorySet` 对象；`CategorySet` 是在 `C++` 层的 `NodeCategorySet::Initialize()` 中导出的，其代码如下

```c++
// node-18.15.0/src/node_trace_events.cc  
void NodeCategorySet::Initialize(Local<Object> target,
                Local<Value> unused,
                Local<Context> context,
                void* priv) {
  
  Local<FunctionTemplate> category_set =
      NewFunctionTemplate(isolate, NodeCategorySet::New); // New方法
  category_set->InstanceTemplate()->SetInternalFieldCount(
      NodeCategorySet::kInternalFieldCount);
  
  // 继承 BaseObject
  category_set->Inherit(BaseObject::GetConstructorTemplate(env));
  
  SetProtoMethod(isolate, category_set, "enable", NodeCategorySet::Enable);
  SetProtoMethod(isolate, category_set, "disable", NodeCategorySet::Disable);

  // 设置CategorySet
  SetConstructorFunction(context, target, "CategorySet", category_set);
  ...
}
```

由代码可知，`JS` 层执行 `new CategorySet(categories)` 实际执行的是 `C++` 层的 `NodeCategorySet::New()` 函数，其代码如下

```c++
// node-18.15.0/src/node_trace_events.cc  
void NodeCategorySet::New(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  ...
  // 创建 NodeCategorySet 对象，args.This 是 CategorySet 对象
  new NodeCategorySet(env, args.This(), std::move(categories));
}
```

`NodeCategorySet::New()` 里创建了一个 `NodeCategorySet` 对象，代码如下

```c++
// node-18.15.0/src/node_trace_events.cc  
class NodeCategorySet : public BaseObject { // NodeCategorySet 继承 BaseObject
  ...
  private:
   NodeCategorySet(Environment* env,
                  Local<Object> wrap,
                  std::set<std::string>&& categories) :
        BaseObject(env, wrap), categories_(std::move(categories)) { // 实例化父类BaseObject
          
    // 调用 BaseObject 的 MakeWeak()  
    MakeWeak();
  }
  
  ...
}

// node-18.15.0/src/base_object.cc 
BaseObject::BaseObject(Realm* realm, Local<Object> object)
    : persistent_handle_(realm->isolate(), object), realm_(realm) { // persistent_handle_ 存储了 object句柄对象（此时是 CategorySet 对象）
  ... 
  SetInternalFields(object, static_cast<void*>(this)); // 设置内部字段引用了C++对象
  ...
}
```

由代码可知，`NodeCategorySet` 类继承 `BaseObject` 类，实例化 `NodeCategorySet` 时也实例化了父类 `BaseObject`， `NodeCategorySet` 子类在构造函数中会调用父类 `BaseObject` 的 `MakeWeak()`。



**所以上面例子的整个垃圾回收过程如下：** 当例子的代码执行完成后， `JS` 层没有变量引用 `createTracing()` 返回的 `Tracing` 对象了，此时 `Tracing` 会被回收 ，因此 `Tracing` 里的 `CategorySet` 对象也会被回收；当 `CategorySet` 被回收后，`V8` 会触发 `BaseObject` 的 `MakeWeak()` 函数设置的 `SetWeak()` 函数，在 `SetWeak()` 回调函数中，`persistent_handle_` 会失去对 `CategorySet` 对象的引用， `C++` 对象 `NodeCategorySet` 也会被删除释放内存资源。



## 2.3 例2

如果对返回的 `Tracing` 对象调用 `enable()` 函数，此时垃圾回收的情况怎样；`enable()` 作用是用于启用跟踪会话，代码如下

```js
setInterval(() => {
    gc();
}, 1000);

const trace_events = require('trace_events');
const events = trace_events.createTracing({categories: ['node.perf', 'node.async_hooks']});

events.enable(); // 调用enable函数
```

因为调用了 `Tracing` 对象的 `enable` 函数，所以 `Tracing` 对象不会被垃圾回收； `enable()` 函数的实现如下

```js
// node-18.15.0/src/trace_events.js
const enabledTracingObjects = new SafeSet();

class Tracing {
  
  constructor(categories) {
    this[kHandle] = new CategorySet(categories);
    this[kCategories] = categories;
    this[kEnabled] = false;
  }


   enable() {
    if (!this[kEnabled]) {
      this[kEnabled] = true;
      this[kHandle].enable(); // 调用 CategorySet的 enable函数
      
      // 把this加入enabledTracingObjects中
      enabledTracingObjects.add(this);
      
      if (enabledTracingObjects.size > kMaxTracingCount) {
        process.emitWarning(
          'Possible trace_events memory leak detected. There are more than ' +
          `${kMaxTracingCount} enabled Tracing objects.`
        );
      }
    }
  }

}  
```

由代码可知，`enable()` 函数会把 `this`（即 `Tracing` 对象） 加入到了 `enabledTracingObjects` 变量，因为 `enabledTracingObjects` 是一直存在的，所以 `Tracing` 对象不会被垃圾回收。



这个例子只有再次通过调用 `Tracing` 对象的 `disable()` 函数才会触发垃圾回收 ，`disable()` 的作用是禁用跟踪会话，它实现如下

```js
// node-18.15.0/src/trace_events.js
class Tracing {
 ...
 disable() {
    if (this[kEnabled]) {
      this[kEnabled] = false;
      this[kHandle].disable();
      enabledTracingObjects.delete(this); // 删除保存的 Tracing 对象
    }
  }
}  
```



# 3 基于引用计数的对象的内存管理机制

前面 `trace_events` 的例子中，关联了 `C++` 对象的 `JS` 层对象是直接暴露给用户的，所以只需要把持有 `JS` 对象的句柄设置为弱持久句柄，并设置弱引用回调，最后在 `JS` 对象被垃圾回收时会释放关联的 `C++` 对象。但是如果这个 `JS` 对象是由 `Node.js` 内核管理，然后通过其他 `API` 来操作这个对象的话，内存管理直接又不一样了。下面以 `diagnostics_channel` 模块为例讲解基于引用计数的对象的内存管理机制



## 3.1 例1

先看下面的例子了解下 `diagnostics_channel` 模块的使用，代码如下

```js
// test.js
const { channel } = require('diagnostics_channel')
const strongRef = channel('strong')

strongRef.subscribe(message => {
  console.log(message) // 会输出
})

channel('weak').subscribe(message => {
  console.log(message) // 注意：这里不会输出，因为channel被回收了
})

setTimeout(() => {
  channel('weak').publish('weak output') // 这里channel('weak')回重新创建一个channel
  strongRef.publish('strong output')
})

gc() // 手动调用gc

// 运行 node --expose-gc test.js  指定--expose-gc才能手动触发gc

// 输出 
// strong output
```

从代码中可知，前面的 `channel('weak')` 创建了一个名字为 `weak` 的 `Channel` 对象，由于我们没有引用返回的 `Channel` 对象，按理此时 `Channel` 对象就会被垃圾回收，实际在这个例子 `Channel` 对象也会被回收；所以后面在 `setTimeout` 中再次执行 `channel('weak')` 时会拿到一个新的 `Channel`；其实这是[一个 bug](https://github.com/nodejs/node/issues/42170)，应该保证前后通过 `channel('weak')` 拿到的`channel` 是同一个对象。这个 `bug` 后面被修复了，[修复方式](https://github.com/nodejs/node/pull/42714)是 `diagnostics_channel` 模块提供了一个新的 `subscribe API`，这个新的 `subscribe` 会增加对 `channel` 对象的引用计数 ，然后旧的 `channel` 对象里的 `subscribe` 方法被标记为过期



## 3.2 例2

使用 `diagnostics_channel` 新的 `subscribe API` 的例子如下

```js
const diagnostics_channel = require('diagnostics_channel')
const { channel } = diagnostics_channel

const strongChannel = channel('strong')

// 这里还是旧的过期的channel类的订阅api
strongChannel.subscribe(message => {
  console.log(message) 
})

// 这里调用新API订阅channel
diagnostics_channel.subscribe('weak', (message) => {
  console.log(message) 
})

setTimeout(() => {
  // 这里channel('weak')拿到的channel跟前面是同一个channel
  channel('weak').publish('weak output') 
  strongChannel.publish('strong output')
})

gc()

// 运行 node --expose-gc test.js  指定--expose-gc才能手动触发gc

// 输出 
// weak output
// strong output
```

思考一下：这个例子的内存回收流程是怎么的？会回收哪些对象？

* 这里用了新的 `subscribe` 去订阅，此时内部实例化的名为 `weak` 的 `Channel` 对象不会被回收了，因为新的 `subscribe()` 函数里面会让 `Channel` 对象被 `C++` 层的 `WeakReference` 所引用；后面在 `setTimeout` 中再次执行 `channel('weak').publish` 时还是会拿到同一个 `Channel`

* 这个例子需要调用 `unsubscribe()` 函数才会触发垃圾回收，此时需要回收的对象如下，`JS` 层自己的 `Channel` 对象（它被传入 `C++` 层，被 `C++` 层引用了），`JS` 层的 `WeakReference` 对象，`C++` 层的 `WeakReference` 对象

> 这里跟前面 `trace_events` 模块的区别是，`diagnostics_channel` 还多了 `C++` 层的 `WeakReference` 对象通过  `_target` 成员变量引用外部 `JS` 层传入的 `Channel` 对象，并通过引用计数来对对象进行内存管理



## 3.3 diagnostics_channel 模块分析

`diagnostics_channel` 的 `JS` 层代码如下

```js
// node-18.15.0/lib/diagnostics_channel.js
const { WeakReference } = internalBinding('util'); // util是node_util.cc

const channels = ObjectCreate(null);

function channel(name) {
  let channel;
  const ref = channels[name];
  if (ref) channel = ref.get();
  if (channel) return channel;

  if (typeof name !== 'string' && typeof name !== 'symbol') {
    throw new ERR_INVALID_ARG_TYPE('channel', ['string', 'symbol'], name);
  }

  channel = new Channel(name); // 创建Channel对象
  channels[name] = new WeakReference(channel); // 创建WeakReference对象
  return channel; // 返回channel
}
```

由代码可知，`channel()` 里会先创建一个 `Channel` 对象，然后以此作为订阅发布机制，接着又创建一个 `WeakReference` 对象，并把 `Channel` 对象传给了 `WeakReference`，最后把 `WeakReference` 对象保存到 `channels` 对象中。`JS` 层拿到的 `WeakReference` 实际是在 `C++` 层 `node_util.cc` 中 `Initialize ` 函数导出的，代码如下

```c++
// node-18.15.0/src/node_util.cc
void Initialize(Local<Object> target,
                Local<Value> unused,
                Local<Context> context,
                void* priv) {
  ...
  Local<FunctionTemplate> weak_ref =
      NewFunctionTemplate(isolate, WeakReference::New); // New 函数
  weak_ref->InstanceTemplate()->SetInternalFieldCount(
      WeakReference::kInternalFieldCount);
  
  // 继承 BaseObject
  weak_ref->Inherit(BaseObject::GetConstructorTemplate(env));
  SetProtoMethod(isolate, weak_ref, "get", WeakReference::Get);
  SetProtoMethod(isolate, weak_ref, "getRef", WeakReference::GetRef);
  SetProtoMethod(isolate, weak_ref, "incRef", WeakReference::IncRef);
  SetProtoMethod(isolate, weak_ref, "decRef", WeakReference::DecRef);
  SetConstructorFunction(context, target, "WeakReference", weak_ref); // WeakReference函数

  ...
}

NODE_BINDING_CONTEXT_AWARE_INTERNAL(util, node::util::Initialize)
```

由代码可知，当在 `JS` 层执行 `new WeakReference(channel)` 时，实际对应执行的是 `C++` 层 `node_util.cc` 的 `WeakReference::New()`，代码如下

```c++
// node-18.15.0/src/node_util.cc
void WeakReference::New(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  CHECK(args.IsConstructCall());
  CHECK(args[0]->IsObject());
  
  // args.This() 为 WeakReference对象
  // args[0] 为 JS 层传入的第一个参数，此时是实例化时传入的Channel对象
  new WeakReference(env, args.This(), args[0].As<Object>());
}

// 接着看 WeakReference 构造函数
// node-18.15.0/src/node_util.cc
WeakReference::WeakReference(Environment* env,
                             Local<Object> object,
                             Local<Object> target)
    : WeakReference(env, object, target, 0) {} // 此时channel对象的引用数传入0

WeakReference::WeakReference(Environment* env,
                             Local<Object> object,
                             Local<Object> target,
                             uint64_t reference_count)
    : SnapshotableObject(env, object, type_int), // 实例化SnapshotableObject
      reference_count_(reference_count) { // 此时channel对象的引用数传入0
        
  // 调用 BaseObject 的  MakeWeak()    
  // 如果 JS 层没有变量引用 new WeakReference 返回的对象，则释放 C++ 的 WeakReference 对象的内存      
  MakeWeak();
        
  if (!target.IsEmpty()) {
    // target_ 是持久句柄，保存对传入的 JS 的对象 Channel 的引用
    target_.Reset(env->isolate(), target);
    if (reference_count_ == 0) {
      // 如果只有 target_ 引用 channel 对象，则该 channel 对象可以被 GC
      target_.SetWeak();
    }
  }
}
```

* `WeakReference` 继承 `SnapshotableObject`，`SnapshotableObject` 继承 `BaseObject`
* `WeakReference` 通过 `reference_count_` 变量记录了 `target` 有多少个引用 （此时 `target` 是 `JS` 层传入的 `Channel` 对象）；接着通过 `WeakReference` 的 `target_` 变量引用 `JS` 层传入的 `Channel` 对象，因为此时 `reference_count_` 是 0，所以执行 `target_.SetWeak()`，表示如果只有 `target_` 自己引用了传入的 `JS` 对象 `Channel`，则该 `JS` 对象 `Channel` 可以被垃圾回收



**subscribe() 实现如下**

注意：这个是 `diagnostics_channel` 模块新加的 `API`，不是旧的 `Channel` 类的 `subscribe API`

```js
// node-18.15.0/lib/diagnostics_channel.js
function subscribe(name, subscription) { 
  const chan = channel(name); // 创建channel
  channels[name].incRef(); // WeakReference::IncRef
  chan.subscribe(subscription);
}
```

由代码可知，通过 `subscribe()` 函数进行订阅后，`subscribe()` 里会调用 `channel()` 函数创建一个 `Channel `对象，如果这个 `channel` 已经存在，会直接拿到同一个 `channel`，然后执行 `incRef()`；`incRef()` 实际对应 `WeakReference::IncRef()` 函数，代码如下

```c++
// node-18.15.0/src/node_util.cc
void WeakReference::IncRef(const FunctionCallbackInfo<Value>& args) {
  WeakReference* weak_ref = Unwrap<WeakReference>(args.Holder());
  
  weak_ref->reference_count_++; // 引用数加1，reference_count_ 变量记录了 channel 有多少个引用
  
  if (weak_ref->target_.IsEmpty()) return;
  if (weak_ref->reference_count_ == 1) weak_ref->target_.ClearWeak(); // 清除弱引用设置，这样Channel才不会被回收
}
```

由代码可知，`WeakReference::IncRef()` 里会对 `Channel` 对象的引用数加一，并且清除弱引用设置，这样保证即使 `JS` 层没有变量引用该 `Channel` 对象，`Channel` 对象也不会被垃圾回收（因为 `C++` 层需要用到该对象，不能先被垃圾回收）。只有显式调用 `unsubscribe()` 函数时才会导致 `Channel` 对象被垃圾回收。



**unsubscribe() 实现如下**

```js
// node-18.15.0/lib/diagnostics_channel.js
function unsubscribe(name, subscription) {
  const chan = channel(name);
  if (!chan.unsubscribe(subscription)) {
    return false;
  }

  channels[name].decRef(); // 引用数减1，对应 WeakReference::DecRef 函数
  
  if (channels[name].getRef() === 0) { // 如果WeakReference的引用计数为0， 则删除 WeakReference 对象
    delete channels[name];
  }
  
  return true;
}

// decRef() 实际对应 WeakReference::DecRef，其代码如下
// node-18.15.0/src/node_util.cc
void WeakReference::DecRef(const FunctionCallbackInfo<Value>& args) {
  WeakReference* weak_ref = Unwrap<WeakReference>(args.Holder());
  CHECK_GE(weak_ref->reference_count_, 1);
  
  weak_ref->reference_count_--; // 引用数减 1
  
  if (weak_ref->target_.IsEmpty()) return;
  if (weak_ref->reference_count_ == 0) weak_ref->target_.SetWeak(); // 如果 reference_count_ 为 0，对引用了Channel对象的句柄设置弱引用回调
}
```

由代码可知，如果 `JS` 层的 `WeakReference` 的引用计数为 0， 则执行 `delete channels[name]` 删除 `JS` 层的 `WeakReference` 对象，最终会触发 `BaseObject` 中 `MakeWeak()` 设置的回调，进而回收 `C++` 层的 `WeakReference` 对象，被 `C++` 层引用的 `JS` 对象 `Channel` 也会被回收。这种就是基于引用计数来对对象进行内存管理的方式。



# 4 基于 HandleWrap 类的内存管理

下面以 `UDPWrap` 类（`HandleWrap` 的子类）为例了解下 `HandleWrap` 类的内存管理机制。`HandleWrap` 是对 `libuv handle` 的封装，它有个 `uv_handle_t*` 类型的 `handle_` 变量，存储一个 `libuv` 的句柄，它指向其子类的 `handle` 结构体，比如 `UDPWrap` 类子类，此时是 `uv_udp_t` 类型，当 `libuv` 句柄关闭时，`HandleWrap` 类需要释放句柄（`uv_udp_t`）所占用的内存（通过显示的调用 `close` 关闭 `handle`）。这种内存管理方式依赖于用户主动关闭 `handle` 才会释放内存。



## 4.1 UDP 模块分析

`JS` 层 `UDP` 的代码如下

```js
// node-18.15.0/lib/dgram.js
const { UDP } = internalBinding('udp_wrap'); // 导入udp_wrap模块(udp_wrap.cc)

function createSocket(type, listener) {
  return new Socket(type, listener);
}

function Socket(type, listener) {
  ...
  const handle = newHandle(type, lookup);
  ...
}

function newHandle(type, lookup) {
  ...
  const handle = new UDP(); // 实例化 UDP，调用的是 udp_wrap.cc 的 UDPWrap::New
  return handle;
}
```

由代码可知，`JS` 层拿到 `udp_wrap` 模块暴露出来的 `UDP` 对象并对它进行实例化。`UDP` 实际是在 `C++` 层 `UDPWrap::Initialize` 函数导出的，其代码如下

```C++
// node-18.15.0/src/upd_wrap.h
// UDPWrap类定义
// 因为UDPWrap继承HandleWrap，HandleWrap继承AsyncWrap，AsyncWrap继承BaseObject，所以UDPWrap继承BaseObject
class UDPWrap final : public HandleWrap,
                      public UDPWrapBase,
                      public UDPListener {
    public:
      // Initialize 定义         
      static void Initialize(v8::Local<v8::Object> target,
                         v8::Local<v8::Value> unused,
                         v8::Local<v8::Context> context,
                         void* priv);
      static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
      ...  
    private:
      uv_udp_t handle_; // 注意：拥有一个libuv的udp句柄，注意这个句柄也要垃圾回收

}

// node-18.15.0/src/upd_wrap.cc
// Initialize实现如下
void UDPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context,
                         void* priv) {
  ...
  Local<FunctionTemplate> t = NewFunctionTemplate(isolate, New); // UDPWrap 的 New 方法

  // 设置一些原型方法
  SetProtoMethod(isolate, t, "open", Open);
  SetProtoMethod(isolate, t, "bind", Bind);
  SetProtoMethod(isolate, t, "connect", Connect);

  ...
    
  // UDPWrap 继承 HandleWrap 类，所以会继承 HandleWrap 的方法，比如 close 方法
  t->Inherit(HandleWrap::GetConstructorTemplate(env));

  ...
}
```

由代码可知，当 `JS` 层执行 `new UDP()` 时，实际对应执行的是 `C++` 层的 `UDPWrap::New()`，其代码如下

```c++
// node-18.15.0/src/udp_wrap.cc
void UDPWrap::New(const FunctionCallbackInfo<Value>& args) {
  ...
  // 实例化 UDPWrap
  new UDPWrap(env, args.This());
}

UDPWrap::UDPWrap(Environment* env, Local<Object> object)
    : HandleWrap(env,
                 object,
                 reinterpret_cast<uv_handle_t*>(&handle_), // 重点：实例化HandleWrap，传入 libuv 句柄，libuv 句柄的 data 属性保存了 JS 对象（UDP）
                 AsyncWrap::PROVIDER_UDPWRAP) {
   // 设置 object 的内置字段，存储当前的 UDPWrap 对象   
   object->SetAlignedPointerInInternalField(
      UDPWrapBase::kUDPWrapBaseField, static_cast<UDPWrapBase*>(this));

  // 注意：初始化 uv_udp_t 句柄    
  int r = uv_udp_init(env->event_loop(), &handle_);
      
  CHECK_EQ(r, 0);  // can't fail anyway

  set_listener(this);
}
```

重点：`UDPWrap` 拥有一个 `libuv` 的 `uv_udp_t` 句柄（`HandleWrap` 有个 `uv_handle_t*` 类型的 `handle_` 变量，它指向其子类的 `handle` 结构体，比如 `UDPWrap` 类是 `HandleWrap` 的子类，此时是 `uv_udp_t` 类型）。当 `libuv` 句柄关闭时，`HandleWrap` 类需要释放句柄（`uv_udp_t`）所占用的内存，并确保与该句柄相关联的 `JS` 对象也被正确地释放，因为该句柄的 `data` 存储了 `JS` 对象

```c++
HandleWrap::HandleWrap(Environment* env,
                       Local<Object> object,
                       uv_handle_t* handle,
                       AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      state_(kInitialized),
      handle_(handle) {
        
  handle_->data = this; // handle_句柄 的 data 拥有 js 对象
        
  HandleScope scope(env->isolate());
  CHECK(env->has_run_bootstrapping_code());
  env->handle_wrap_queue()->PushBack(this);
}
```



**UDPWrap 对象和 JS 对象的引用关系**

从 `UDPWrap` 类的定义可知，`UDPWrap` 继承 `HandleWrap`，`HandleWrap` 继承 `AsyncWrap`，`AsyncWrap` 继承`BaseObject`，所以 `UDPWrap` 最终也继承 `BaseObject`。`BaseObject` 构造函数内会关联 `JS` 和 `C++` 对象，`BaseObject` 类的构造函数代码如下

```c++
// node-18.15.0/src/base_object.cc
BaseObject::BaseObject(Realm* realm, Local<Object> object)
    : persistent_handle_(realm->isolate(), object), realm_(realm) {
  ...
  // 把 this（UDPWrap对象）存到 object中
  SetInternalFields(object, static_cast<void*>(this));
  // 退出释放当前 this 对象的内存
  realm->AddCleanupHook(DeleteMe, static_cast<void*>(this));
}
```



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/udpWrap.png)



由代码可知，`C++` 层通过持久句柄 `persistent_handle_` 引用了 `UDPWrap` 对象，即使 `JS` 层没有变量引用该对象，该对象也不会被垃圾回收（因为 `UDPWrap` 在实例化过程中不会调用 `BaseObject` 的 `MakeWeak()` 函数去设置弱引用回调）。只有当在 `JS` 层主动调用 `Socket` 类的 `close()` 时才会回收 `UDPWrap` 对象



## 4.2 从 Socket::close 看内存管理

只有当调用执行完 `Socket` 的 `close()` 时， `UDPWrap` 对象才会被垃圾回收。`UDPWrap` 拥有一个 `libuv` 的 `uv_udp_t` 句柄（`HandleWrap` 有个 `uv_handle_t*` 类型的 `handle_` 变量，它指向其子类的 `handle` 结构体，比如 `UDPWrap` 类是 `HandleWrap` 的子类，此时是 `uv_udp_t` 类型）。当 `libuv` 句柄关闭时，`HandleWrap` 类需要释放句柄（`uv_udp_t`）所占用的内存，并确保与该句柄相关联的 `JS` 对象（`UDPWrap`）也被正确地释放，因为该句柄的 `data` 存储了 `JS` 对象（`UDPWrap`）



### 4.2.1 Socket.close() 函数

通过例子看一下 `close()` 的使用

```js
// test.js
const dgram = require('dgram');
const socket = dgram.createSocket('udp4');

socket.close(); 

setInterval(() => {
    gc();
    console.log('手动触发gc');
}, 1000);

// 运行 node --expose-gc test.js
```



**Socket 的 close() 函数实现如下**

```js
// node-18.15.0/lib/dgram.js
Socket.prototype.close = function(callback) {
  const state = this[kStateSymbol];
  ...
  state.handle.close(); // handle即UDP对象
  state.handle = null;
  ...
}
  
// node-18.15.0/lib/dgram.js
function newHandle(type, lookup) {
  ...
  const handle = new UDP(); // handle 即 UDP 对象
  return handle;
}
  
// node-18.15.0/src/upd_wrap.cc
void UDPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context,
                         void* priv) {
  ...
  // UDPWrap 继承 HandleWrap 类，所以会继承HandleWrap的方法，比如 close 方法
  t->Inherit(HandleWrap::GetConstructorTemplate(env));
  ...
}
```

由代码可知，`close()` 内部会先调用 `handle` 的 `close()` 函数，然后再把 `handle` 置为 `null`。因为 `UDP` 继承了 `HandleWrap` 类，这里 `handle`  就是 `UPD` 对象， 所以 `handle` 的 `close()` 实际就是 `HandleWrap` 类的` Close()` 函数



### 4.2.2 HandleWrap::Close() 函数

`HandleWrap::Close()` 函数的实现如下

```c++
// node-18.15.0/src/handle_wrap.cc
void HandleWrap::Close(const FunctionCallbackInfo<Value>& args) {
  HandleWrap* wrap;
  
  // 从js对象拿到c++对象
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

  // 继续调用Close()函数
  wrap->Close(args[0]); // args[0]为close函数传入的callback函数，这里没有传
}

void HandleWrap::Close(Local<Value> close_callback) {
  ...
  // 传入句柄对象
  // 调用libuv的uv_close，OnClose 是回调函数，用于在句柄关闭后执行一些清理操作
  uv_close(handle_, OnClose);
  ...
}

// libuv 代码如下

// node-18.15.0/deps/uv/src/unix/core.c
void uv_close(uv_handle_t* handle, uv_close_cb close_cb) {
  ...
    
  handle->flags |= UV_HANDLE_CLOSING;
  handle->close_cb = close_cb; // C++层的OnClose函数
  
  switch (handle->type) {
  case UV_UDP:
    uv__udp_close((uv_udp_t*)handle);
    break;
  }
      
}

void uv__udp_close(uv_udp_t* handle) {
  uv__io_close(handle->loop, &handle->io_watcher);
  uv__handle_stop(handle);

  if (handle->io_watcher.fd != -1) {
    uv__close(handle->io_watcher.fd);
    handle->io_watcher.fd = -1;
  }
}
```

由代码可知，`HandleWrap` 类的 `Close()` 函数实际调用的是 `libuv` 的 `uv_close()`，`uv_close()` 传入了`OnClose` 回调函数，该 `OnClose` 函数会在事件循环的 `close` 阶段执行，用于在句柄关闭后执行一些清理操作。



### 4.2.3 HandleWrap::OnClose() 回调函数

`HandleWrap::OnClose()` 回调函数的实现如下

```c++
// node-18.15.0/src/handle_wrap.cc
void HandleWrap::OnClose(uv_handle_t* handle) {
  ...
 
  // handle-> data 表示取到 HandleWrap 对象，所以转成 HandleWrap* 类型
  // 这里 BaseObjectPtr 用于管理 HandleWrap，BaseObjectPtr 把 HandleWrap 封装存储到 BaseObjectPtr 的成员变量中
  BaseObjectPtr<HandleWrap> wrap { static_cast<HandleWrap*>(handle->data) };
  
  // 用于从句柄队列中移除该句柄，并断开句柄与HandleWrap对象的关联
  wrap->Detach();
  
  ...
  
  // 检查 HandleWrap 对象的状态是否为 kClosing，如果不是则直接返回
  CHECK_EQ(wrap->state_, kClosing);

  // 将 HandleWrap 对象的状态设置为 kClosed，表示该对象已经关闭
  wrap->state_ = kClosed;

  wrap->OnClose(); // 调用 HandleWrap 对象的 OnClose() 函数，执行对象关闭时的逻辑
  
  wrap->handle_wrap_queue_.Remove(); // 将 HandleWrap 对象从 HandleWrapQueue 队列中移除

  // 如果 HandleWrap 对象的持久化句柄不为空，并且对象上存在 handle.onclose 回调函数，则调用该回调函数
  if (!wrap->persistent().IsEmpty() &&
      wrap->object()->Has(env->context(), env->handle_onclose_symbol())
      .FromMaybe(false)) {
    wrap->MakeCallback(env->handle_onclose_symbol(), 0, nullptr);
  }
}

// handle->data 是在实例化 HandleWrap 时保存的，如下
HandleWrap::HandleWrap(Environment* env,
                       Local<Object> object, // object 为 JS 对象 
                       uv_handle_t* handle, // handle 为子类具体的 handle 类型，不同模块不一样 ，此时为 uv_udp_t
                       AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      state_(kInitialized),
      handle_(handle) { // 实例化handle_

  // 保存 libuv handle 和 C++ 对象的关系，libuv 执行 C++ 回调时使用
  handle_->data = this;
  ...      
}

// Detach() 函数将对象的 is_detached 标记设为 true，用于解除对象与句柄之间的关联
// node-18.15.0/src/base_object-inl.h
void BaseObject::Detach() {
  // 检查对象的强引用计数（strong_ptr_count）的值是否大于 0，如果不是，则表示对象已经被释放，无法再进行分离操作，直接返回
   CHECK_GT(pointer_data()->strong_ptr_count, 0);

  // 将对象的 is_detached 标记设为 true，表示该对象已经被分离
  pointer_data()->is_detached = true;
}
```

`BaseObjectPtr` 类是用于管理 `C++` 对象的智能指针类。它是 `BaseObjectPtrImpl` 模板类的一个别名，用来管理 `C++` 对象的内存（这里是用于管理 `HandleWrap` 对象），它提供了自动内存管理功能，可以有效避免内存泄漏和安全问题。`BaseObjectPtr` 类中的智能指针是基于引用计数的机制实现的。`BaseObjectPtrImpl` 代码如下：

```c++
// node-18.15.0/src/base_object.h
// BaseObjectPtrImpl类的data_成员属性
template <typename T, bool kIsWeak>
class BaseObjectPtrImpl final {
  ...
  private:
    union {
      BaseObject* target;                     // Used for strong pointers.
      BaseObject::PointerData* pointer_data;  // Used for weak pointers.
    } data_;
  ...
}

// node-18.15.0/src/base_object-inl.h

// 为BaseObjectPtrImpl<T, false> 起别名为BaseObjectPtr，此时kIsWeak值为false
using BaseObjectPtr = BaseObjectPtrImpl<T, false>;

// BaseObjectPtrImpl 默认构造函数的实现
template <typename T, bool kIsWeak>
BaseObjectPtrImpl<T, kIsWeak>::BaseObjectPtrImpl() {
  data_.target = nullptr; // 初始化data_的target
}

template <typename T, bool kIsWeak>
BaseObjectPtrImpl<T, kIsWeak>::BaseObjectPtrImpl(T* target)
  : BaseObjectPtrImpl() {
    
  if (target == nullptr) return;
    
  if constexpr (kIsWeak) { // 如果是弱引用，实例化 BaseObjectPtrImpl 的 pointer_data 属性
    data_.pointer_data = target->pointer_data();
    ...
    pointer_data()->weak_ptr_count++; // 增加计数
  } else { // 如果是强引用
    data_.target = target; // 实例化 BaseObjectPtrImpl 的 target 属性
    ...
    get()->increase_refcount(); // 增加引用计数
  }
}

// node-18.15.0/src/base_object.cc
// 增加自定义对象的引用计数 (此时是 HandleWrap )
void BaseObject::increase_refcount() {
  // 获取自定义对象的指针数据把强引用计数加1
  unsigned int prev_refcount = pointer_data()->strong_ptr_count++;
  
  // 该对象的强引用计数从 0 变为 1，所以需要调用 ClearWeak() 清除对象的弱引用标记，防止被 GC
  if (prev_refcount == 0 && !persistent_handle_.IsEmpty())
    persistent_handle_.ClearWeak();
}


```

`BaseObjectPtrImpl` 类中包含一个 `data_` 成员变量，它是一个联合体（`union`），包含两个成员：`target` 和 `pointer_data`。`target` 成员用于存储强引用指针，`pointer_data` 成员用于存储弱引用指针。当使用强引用指针时，将 `C++` 对象的指针存储在 `target` 成员中。当使用弱引用指针时，将一个指向 `PointerData` 对象的指针存储在 `pointer_data` 成员中，`PointerData` 对象中包含了一个指向 `C++` 对象的指针。



在执行完 `HandleWrap::OnClose()` 函数后，  `BaseObjectPtr` 对象会被析构，代码如下

```c++
// node-18.15.0/src/base_object-inl.h
template <typename T, bool kIsWeak>
BaseObjectPtrImpl<T, kIsWeak>::~BaseObjectPtrImpl() {
  ...
  // get() 返回 BaseObject*
  get()->decrease_refcount();
  ...
}

// node-18.15.0/src/base_object.cc
// 减少对象的强引用计数，并在计数为 0 时执行清理逻辑
void BaseObject::decrease_refcount() {
  ...
  // 获取对象的指针数据
  PointerData* metadata = pointer_data();
  
  // 将对象的强引用计数减 1
  unsigned int new_refcount = --metadata->strong_ptr_count;
  if (new_refcount == 0) { // 对象的强引用计数变为 0 则执行清理逻辑
    if (metadata->is_detached) { // true表示对象已经被分离，则调用 OnGCCollect() 函数执行对象的垃圾回收
      OnGCCollect(); // OnGCCollect 最终执行 delete this;
    // 如果对象希望拥有弱引用的js对象（wants_weak_jsobj 为 true），并且持久化句柄不为空
    } else if (metadata->wants_weak_jsobj && !persistent_handle_.IsEmpty()) {
      MakeWeak(); // 将持久化句柄转换为弱引用句柄，用于在 js 对象被垃圾回收时释放 C++ 对象的内存
    }
  }
}

// 执行对象的垃圾回收逻辑，并释放C++对象的内存
void BaseObject::OnGCCollect() {
  delete this;
}
```

`BaseObjectPtr` 析构后就会释放 `this` 指针指向的内存（`HandleWrap`对象），当 `V8` 中不存在任何对 `BaseObject` 对象的引用时，`persistent_handle_` 对象会被 `V8` 的垃圾回收机制回收，从而 `BaseObject` 对象的`persistent_handle_` 属性（`Global` 类型）也被析构

```c++
// node-18.15.0/deps/v8/include/v8-persistent-handle.h
// 用于释放 Global 对象所持有的资源，并执行对象的清理逻辑
V8_INLINE ~Global() { this->Reset(); }

// 注意：Global 对象通常用于在 JavaScript 中创建全局变量，因此它所持有的资源包括了 V8 引擎的上下文（Context）对象和 JavaScript 中的全局变量对象。在调用 Reset() 函数时，会释放这些资源，并将 Global 对象重置为初始状态，即没有关联的上下文和全局变量
```

在 `Global()` 析构函数中，`Reset()` 使得 `persistent_handle_` 不再引用 `JS` 对象，最终 `JS` 对象失去了所有的引用，从而 `JS` 对象也被回收。



# 5 基于 ReqWrap 类的内存管理

下面以 `ConnectWrap` 类（`ReqWrap` 的子类） 为例了解 `ReqWrap` 类的内存管理机制。`RewWrap` 是对请求的封装，从发起连接到结束连接整个过程都是由 `Node.js` 自动管理内存



## 5.1 从建立 TCP 连接看内存管理

### 5.1.1 net.connect() 函数

`Node.js` 中，可以使用 [net.connect()](https://nodejs.org/docs/v18.15.0/api/net.html#netconnect) 或 [net.createConnection()](https://nodejs.org/docs/v18.15.0/api/net.html#netcreateconnection) 建立一个 `TCP` 连接，例如

```js
// test.js
const net = require('net');

setInterval(() => {
    gc();
    console.log('手动触发gc');
}, 2000);

setTimeout(() => {
    // 建立连接
    net.connect(8888, '127.0.0.1').on('error', () => {});
}, 1000);

// 运行 node --expose-gc test.js 指定--expose-gc才能手动触发gc
```



**JS 层的 net.connect() 函数如下**

```js
// node-18.15.0/lib/net.js
const {
  TCP,
  TCPConnectWrap
} = internalBinding('tcp_wrap');

function connect(...args) {
  const normalized = normalizeArgs(args);
  const options = normalized[0];
  debug('createConnection', normalized);
  
  // 实例化Socket
  // 如果是tcp类型，则 Socket对象的_handle属性是一个TCP对象，即new TCP(is_server ? TCPConstants.SERVER : TCPConstants.SOCKET);
  const socket = new Socket(options); 
  
  lazyChannels();
  if (netClientSocketChannel.hasSubscribers) {
    netClientSocketChannel.publish({
      socket,
    });
  }
  if (options.timeout) {
    socket.setTimeout(options.timeout);
  }

  // 调用了Socket的connect函数，实际调用了 c++ 层 tcp_wrap.cc 的 Connect()
  return socket.connect(normalized);
}

// connect 调用了 internalConnect
Socket.prototype.connect = function(...args) {
  ...
  if (pipe) {
    validateString(path, 'options.path');
    defaultTriggerAsyncIdScope( // 里面调用 internalConnect
      this[async_id_symbol], internalConnect, this, path
    );
  } else {
    lookupAndConnect(this, options); // 里面调用 internalConnect
  }
  return this;
}

// internalConnect 中实例化了 TCPConnectWrap，然后调用self._handle.connect，实际是调用了C++层的TCPWrap::Connect
function internalConnect(
  self, address, port, addressType, localAddress, localPort, flags) {
  ...
  
  // 实例化 TCPConnectWrap
  const req = new TCPConnectWrap();
    
  req.oncomplete = afterConnect;
  req.address = address;
  req.port = port;
  req.localAddress = localAddress;
  req.localPort = localPort;

  if (addressType === 4)
    err = self._handle.connect(req, address, port); // _handle 为 new TCP 返回的对象，实际调用的是 c++ 层 TCPWrap::Connect
    
  // 执行完后 JS 层会失去对 TCPConnectWrap 的引用
  
}

module.exports = {
  connect,
  createConnection: connect, // createConnection其实也是connect
}
```

由代码可知，`JS` 层使用的 `TCP` 和 `TCPConnectWrap` 是在 `TCPWrap::Initialize()` 函数中暴露出去的，代码如下

```c++
// node-18.15.0/src/tcp_wrap.cc
void TCPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context,
                         void* priv) {
  ...
  // 为TCP创建一个函数模板  
  Local<FunctionTemplate> t = NewFunctionTemplate(isolate, New);
  SetProtoMethod(isolate, t, "connect", Connect); // TCPWrap 的 Connect 方法
  SetConstructorFunction(context, target, "TCP", t);

  
  // 为 TCPConnectWrap 创建一个函数模板
  Local<FunctionTemplate> cwt =
      BaseObject::MakeLazilyInitializedJSTemplate(env);
  // 继承 AsyncWrap
  cwt->Inherit(AsyncWrap::GetConstructorTemplate(env));
  // 设置构造方法 TCPConnectWrap
  SetConstructorFunction(context, target, "TCPConnectWrap", cwt);
  ...
}
```

由代码可知，`JS` 层执行 `new TCPConnectWrap(xx)` 实际实例化的是 `AsyncWrap` 类，`JS` 层的 `connect()` 函数实际对应 `C++` 层的是 `TCPWrap::Connect()`



### 5.1.2 TCPWrap::Connect() 函数

`C++` 层的是 `TCPWrap::Connect()` 函数代码如下

```c++
// node-18.15.0/src/tcp_wrap.cc
template <typename T>
void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args,
    std::function<int(const char* ip_address, T* addr)> uv_ip_addr) {
  Environment* env = Environment::GetCurrent(args);
  ...
  Local<Object> req_wrap_obj = args[0].As<Object>(); // 即 JS 层的 TCPConnectWrap 对象

  ...
  // 实例化一个 ConnectWrap 对象
  ConnectWrap* req_wrap =
        new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);

  // 调用 libuv的uv_tcp_connect函数
  err = req_wrap->Dispatch(uv_tcp_connect,
                             &wrap->handle_,
                             reinterpret_cast<const sockaddr*>(&addr),
                             AfterConnect);
  
  if (err) {
      delete req_wrap;
  }
}

// node-18.15.0/src/connect_wrap.cc
// 实例化ConnectWrap的实现
ConnectWrap::ConnectWrap(Environment* env,
    Local<Object> req_wrap_obj,
    AsyncWrap::ProviderType provider) : ReqW
```

由代码可知，`TCPWrap::Connect` 里主要有两个操作

1. 实例化 `ConnectWrap` 对象
2. 通过 `req_wrap->Dispatch` 调用 `libuv` 的 `uv_tcp_connect` 函数



**实例化 ConnectWrap 对象**

 `TCPWrap::Connect` 里面实例化了 `ConnectWrap` 对象，此时把 `JS` 层传进来的 `TCPConnectWrap` 对象 和 `C++` 层的 `ConnectWrap` 对象关联起来

```c++
// 实例化ConnectWrap
ConnectWrap* req_wrap =
       new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);
```

因为 `ConnectWrap` 继承 `ReqWrap` 类，`ReqWrap` 继承 `AsyncWrap` 类，`AsyncWrap` 继承 `BaseObject` 类。所以实例化 `ConnectWrap` 类时，也会实例化 `ReqWrap` ，`ReqWrap` 构造函数会调用父类 `BaseObject` 的 `MakeWeak() `函数，代码如下

```c++
// node-18.15.0/src/req_wrap-inl.h
template <typename T>
ReqWrap<T>::ReqWrap(Environment* env,
                    v8::Local<v8::Object> object,
                    AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      ReqWrapBase(env) {
  MakeWeak(); // MakeWeak 用于给持久引用设置弱引用回调，实际是父类 BaseObject的方法
  Reset();
}

// node-18.15.0/src/base_object.cc
void BaseObject::MakeWeak() {
  if (has_pointer_data()) {
    pointer_data()->wants_weak_jsobj = true;
    if (pointer_data()->strong_ptr_count > 0) return;
  }

  // 设置为弱持久句柄
  persistent_handle_.SetWeak(
      this,
      [](const WeakCallbackInfo<BaseObject>& data) {
        BaseObject* obj = data.GetParameter();
        // Clear the persistent handle so that ~BaseObject() doesn't attempt
        // to mess with internal fields, since the JS object may have
        // transitioned into an invalid state.
        // Refs: https://github.com/nodejs/node/issues/18897
        obj->persistent_handle_.Reset();
        CHECK_IMPLIES(obj->has_pointer_data(),
                      obj->pointer_data()->strong_ptr_count == 0);
        obj->OnGCCollect();
      },
      WeakCallbackType::kParameter);
}
```



**调用 libuv的uv_tcp_connect函数**

实例化 `ConnectWrap` 对象后，接着通过 `req_wrap->Dispatch(xx)` 调用 `libuv` 的 `uv_tcp_connect()` 函数，代码如下

```c++
// node-18.15.0/src/tcp_wrap.cc
void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args,
    std::function<int(const char* ip_address, T* addr)> uv_ip_addr) {
  ...

  // 调用 libuv的uv_tcp_connect函数
  err = req_wrap->Dispatch(uv_tcp_connect,
                             &wrap->handle_,
                             reinterpret_cast<const sockaddr*>(&addr),
                             AfterConnect);
  
  // 0 表示TCP连接请求已经成功发送到对端，但是连接的建立过程还未完成，需要等待后续事件通知
  // 正数表示执行失败
  // 负数表示在调用uv_tcp_connect函数时发生了错误
  if (err) { // 表示失败
      delete req_wrap;
  }
}

// Dispatch的实现
// node-18.15.0/src/req_wrap-inl.h
int ReqWrap<T>::Dispatch(LibuvFunction fn, Args... args) {
  Dispatched();
  // This expands as:
  //
  // int err = fn(env()->event_loop(), req(), arg1, arg2, Wrapper, arg3, ...)
  //              ^                                       ^        ^
  //              |                                       |        |
  //              \-- Omitted if `fn` has no              |        |
  //                  first `uv_loop_t*` argument         |        |
  //                                                      |        |
  //        A function callback whose first argument      |        |
  //        matches the libuv request type is replaced ---/        |
  //        by the `Wrapper` method defined above                  |
  //                                                               |
  //               Other (non-function) arguments are passed  -----/
  //               through verbatim
  // 调用 libuv 函数
  int err = CallLibuvFunction<T, LibuvFunction>::Call(
      fn,
      env()->event_loop(),
      req(),
      MakeLibuvRequestCallback<T, Args>::For(this, args)...);
  
  // 正数 表示执行失败
  // 0 表示TCP连接请求已经成功发送到对端，但是连接的建立过程还未完成，需要等待后续事件通知
  if (err >= 0) {
    ClearWeak(); // BaseObject的方法
    env()->IncreaseWaitingRequestCounter();
  }
  
  return err;
}
```

由代码可知，`ReqWrap<T>::Dispatch` 里会通过 `CallLibuvFunction` 去调用 `libuv` 的函数，其中 `MakeLibuvRequestCallback` 的 `For` 函数是用于创建一个 `libuv` 的回调函数。 `err >= 0` 会调用 `ClearWeak()` 函数，`err == 0` 表示连接成功，`err > 0` 表示执行失败时，代码如下

```c++
if (err >= 0) {
    ClearWeak(); // BaseObject的方法
    env()->IncreaseWaitingRequestCounter();
 }

// node-18.15.0/src/base_object-inl.h
void BaseObject::ClearWeak() {
  if (has_pointer_data())
    pointer_data()->wants_weak_jsobj = false;

  // 取消弱持久句柄，变成一般持久句柄
  persistent_handle_.ClearWeak(); 
}
```

`ClearWeak()` 的作用是取消 `persistent_handle_` 句柄的弱持久标记，使其变成一般持久句柄（`persistent_handle_ ` 是在 `ReqWrap` 构造函数的 `MakeWeak()` 设置成了弱持久句柄）。之所以要变成一般持久句柄，是因为需要让 `JS` 对象（ `TCPConnectWrap` ）被 `ConnectWrap` 类的持久句柄 `persistent_handle_` 引用，才不会导致 `JS` 对象（ `TCPConnectWrap` ）被回收，因为连接是一个异步操作，在等待连接结果的过程中 `JS` 对象（`TCPConnectWrap`）被回收了会导致进程崩溃。

> 也因为 `persistent_handle_` 被变成了一般持久句柄，后面需要显示的调用 `Reset()` 函数重置句柄所引用的 `JS `对象，才能使得 `JS` 对象失去该句柄的唯一引用，从而被回收



当 `Dispatch` 执行结束时，`Dispatch` 会返回 `err`，代码如下

```c++
if (err) { // 表示失败
      delete req_wrap; // 删除 ConnectWrap 对象
 }
```

当 `err` 为真时，说明调用 `libuv` 的 `uv_tcp_connect` 失败了，此时会删除 `ConnectWrap` 对象， 并释放 `ConnectWrap` 对象的内存，此时会依次调用 `ConnectWrap` 及其父类的析构函数，即依次调用 `ConnectWrap` 类、 `AsyncWrap` 类、`BaseObject` 类的析构函数。`BaseObject` 的 `persistent_handle_` 对象的析构函数是在 `BaseObject` 对象被销毁时自动调用的，它会释放对 `JS` 层对象（`TCPConnectWrap`）的引用，并释放 `persistent_handle_` 对象自己的内存；因为 `JS` 对象（`TCPConnectWrap`） 将失去 `persistent_handle_` 这个唯一的引用，最后 `TCPConnectWrap` 对象也被 `V8` 回收。

> `persistent_handle_` 对象的析构函数会调用句柄的 `Reset()`，`Reset()` 函数可以将句柄对象的引用重置为 `undefined` 或 `null`，从而释放句柄对象所持有的 `JS` 对象的引用

```c++
// node-18.15.0/deps/v8/include/v8-persistent-handle.h
// persistent_handle_ 的析构函数，用于释放 Global 对象所持有的资源，并执行对象的清理逻辑
V8_INLINE ~Global() { this->Reset(); }

```



**MakeLibuvRequestCallback::For 函数**

`MakeLibuvRequestCallback::For` 的作用是创建一个回调函数，将异步回调处理的代码封装在一个通用的模板类中，用于在 `libuv` 事件循环中异步执行某个操作完成后的回调处理，从而使得可以在不同的场景中使用相同的代码，实现如下

```c++
// node-18.15.0/src/req_wrap-inl.h
template <typename ReqT, typename... Args>
struct MakeLibuvRequestCallback<ReqT, void(*)(ReqT*, Args...)> {
  // 定义一个类型别名F
  using F = void(*)(ReqT* req, Args... args);

  static void Wrapper(ReqT* req, Args... args) {
    // BaseObjectPtr 构造函数导致 C++ 对象的引用数加一
    BaseObjectPtr<ReqWrap<ReqT>> req_wrap{ReqWrap<ReqT>::from_req(req)};
    
    req_wrap->Detach();
    
    req_wrap->env()->DecreaseWaitingRequestCounter();
    
    // 取到原始回调函数
    F original_callback = reinterpret_cast<F>(req_wrap->original_callback_);
    
    // 执行回调函数
    original_callback(req, args...);
  }

  // 静态成员For函数
  // For函数的作用是在当前对象上创建一个回调函数，当libuv事件循环中的异步操作完成后，会调用这个回调函数来处理异步操作的结果
  static F For(ReqWrap<ReqT>* req_wrap, F v) {
    CHECK_NULL(req_wrap->original_callback_);
    
    // 保存了原始回调函数 v
    req_wrap->original_callback_ =
        reinterpret_cast<typename ReqWrap<ReqT>::callback_t>(v);
    
    // 返回Wrapper函数
    return Wrapper;
  }
};

template <typename T>
void ReqWrap<T>::Dispatched() {
  req_.data = this;
}

```

`For` 函数保存了原始回调，然后返回一个 `Wrapper` 函数，当 `libuv` 处理完执行回调时就会执行 `Wrapper`，`Wrapper` 中通过拿到 `ConnectWrap` 对象（`ConnectWrap` 是 `ReqWrap` 的子类） ，并用一个智能指针 类`BaseObjectPtr` 管理 `ReqWrap` 对象，此时 `ConnectWrap` 引用数加一，然后调用 `Detach` 设置一个 `detach` 标记，调完 `Detach` 后就执行真正的回调函数 `original_callback`，这里回调函数是 `ConnectionWrap` 的 `AfterConnect` 函数，其实现如下

```c++
// node-18.15.0/src/connection_wrap.cc
template <typename WrapType, typename UVType>
void ConnectionWrap<WrapType, UVType>::AfterConnect(uv_connect_t* req,
                                                    int status) {
  
  BaseObjectPtr<ConnectWrap> req_wrap{static_cast<ConnectWrap*>(req->data)};
  
  ...
    
  // 执行 JS 回调
  req_wrap->MakeCallback(env->oncomplete_string(), arraysize(argv), argv);
}
```

`AfterConnect` 中定义了一个 `BaseObjectPtr`，所以这时 `C++` 对象 `ConnectWrap` 的引用数加 1 后变成 2，接着执行了 `JS` 层回调函数，执行完之后，`Wrapper` 和 `AfterConnect` 函数中的 `BaseObjectPtr` 会析构，从而 `ConnectWrap` 对象被析构，最终 `BaseObject` 的持久句柄会被析构， `TCPConnectWrap` 对象失去最后一个引用而被垃圾回收（如果把该对象返回给 `JS` 层使用，则会在 `JS` 层失去引用后再被 `GC`）。这种方式对用户是无感知的，完全由 `Node.js` 控制内存的使用和释放。



# 6 Node.js 提供的内存管理机制: ObjectWrap类

`Node.js` 内部除了提供 `BaseObject` 管理对象的内存之外，还提供了 `ObjectWrap` 类，使得我们只要继承该类就可以使用 `Node.js` 提供的内存管理机制。



## 6.1 ObjectWrap 类的实现

`ObjectWrap` 提供了 `JS` 和 `C++` 对象的生命周期管理，只要继承该类就可以实现内存管理机制；`ObjectWrap` 实现如下

```c++
// node-18.15.0/src/node_object_wrap.h
class ObjectWrap {
 public:
  ObjectWrap() {
    refs_ = 0; // 引用数
  }
  
   virtual ~ObjectWrap() {
     if (persistent().IsEmpty())
       return;
     persistent().ClearWeak();
     persistent().Reset();
  }

  template <class T>
  static inline T* Unwrap(v8::Local<v8::Object> handle) {
    assert(!handle.IsEmpty());
    assert(handle->InternalFieldCount() > 0);
    // Cast to ObjectWrap before casting to T.  A direct cast from void
    // to T won't work right when T has more than one base class.
    void* ptr = handle->GetAlignedPointerFromInternalField(0);
    ObjectWrap* wrap = static_cast<ObjectWrap*>(ptr);
    return static_cast<T*>(wrap);
  }

  inline v8::Local<v8::Object> handle() {
    return handle(v8::Isolate::GetCurrent());
  }

  inline v8::Local<v8::Object> handle(v8::Isolate* isolate) {
    return v8::Local<v8::Object>::New(isolate, persistent());
  }

  // NOLINTNEXTLINE(runtime/v8_persistent)
  inline v8::Persistent<v8::Object>& persistent() {
    return handle_;
  }

 protected:
  inline void Wrap(v8::Local<v8::Object> handle) {
    assert(persistent().IsEmpty());
    assert(handle->InternalFieldCount() > 0);
    
    // 关联 JS 和 C++ 对象，用 JS 的内置字段存储 C++ 对象
    handle->SetAlignedPointerInInternalField(0, this);
    persistent().Reset(v8::Isolate::GetCurrent(), handle);
    
    // 默认设置了弱引用，如果 JS 对象没有被其他变量引用则会被垃圾回收
    MakeWeak();
  }

  // 设置弱引用回调
  inline void MakeWeak() {
    persistent().SetWeak(this, WeakCallback, v8::WeakCallbackType::kParameter);
  }
  
  // 引用数加 1，清除弱引用回调
  virtual void Ref() {
    assert(!persistent().IsEmpty());
    persistent().ClearWeak();
    refs_++;
  }
  
  // 引用数减 1，设置弱引用回调
  virtual void Unref() {
    ...
    if (--refs_ == 0)
      MakeWeak();
  }

  int refs_; // 引用数

 private:
  // 弱引用回调函数
  static void WeakCallback(
      const v8::WeakCallbackInfo<ObjectWrap>& data) {
    ObjectWrap* wrap = data.GetParameter();
    assert(wrap->refs_ == 0);

    // 解除引用 JS 对象
    wrap->handle_.Reset();
    
    // 删除对象，释放内存
    delete wrap;
  }

  // 通过持久引用保存 JS 对象，避免被 JS 对象被 GC
  v8::Persistent<v8::Object> handle_;
};
```

注意：因为 `ObjectWrap` 默认设置了弱引用，如果管理的 `JS` 对象没有被其他变量引用则会被垃圾回收，可以主动调用 `Ref()` 改变这个行为。`ObjectWrap` 的使用例子可以参考[这里](https://github.com/sinkhaha/node-addon-demo/tree/main/object-wrap-demo)



## 6.2 追踪 JS 对象是否被 GC

应用场景：可以利用这种弱引用机制追踪 `JS` 对象是否被 `GC`。



想追踪一个 `JS` 对象是否被 `GC` 时，就可以把该 `JS` 对象作为键保存在 `WeakMap` 中，值存储一个特殊的值，这个值的特殊之处在于当键被 `GC` 时，值也会被 `GC`，再通过给值设置弱引用回调得到通知，也就是说当回调被执行时，说明值被 `GC` 了，也就说明键被 `GC` 了，这个值的类型是 `AsyncResource`。`AsyncResource` 帮我们处理好了底层的事件，我们只需要通过 `async_hooks` 的 `destroy` 钩子就可以知道哪个 `AsyncResource` 被 `GC` 了，从而知道哪个键被 `GC` 了，最后执行一个回调。

> 这里也利用了 `WeakMap`，`WeakMap` 键只能存储对象且 `WeakMap` 对该对象是弱引用，即如果没有其他变量引用该对象则该对象会被垃圾回收，并且值也会被垃圾回收。



**实现代码如下**

```js
// 新建 index.js
const { createHook, AsyncResource } = require('async_hooks');
const weakMap = new WeakMap(); // key是要被追踪的JS对象，value 是AsyncResource对象

// 存储被监控对象和 GC 回调的映射
const gcCallbackContext = {};

let hooks;

// 追踪对象，并设置回调函数
function trackGC(obj, gcCallback) {
  if (!hooks) {
    // 创建一个AsyncHook，并注册destroy钩子函数
    hooks = createHook({
      destroy(asyncId) {
        if (gcCallbackContext[asyncId]) {
          gcCallbackContext[asyncId](); // 执行回调
          delete gcCallbackContext[asyncId]; // 删除回调函数
        }
      }
    }).enable();
  }
  
  const gcTracker = new AsyncResource('none');
  
  // 通过 asyncId 记录被追踪对象和 GC 回调的映射
  gcCallbackContext[gcTracker.asyncId()] = gcCallback;
  
  // 设置js对象跟 AsyncResource 的关系
  weakMap.set(obj, gcTracker);
}

module.exports = {
    trackGC
}
```

```js
// 新建 test.js
const { trackGC } = require('./index'); // 导入index

function memory() {
  return ~~(process.memoryUsage().heapUsed / 1024 / 1024);
}

console.log(`before new Array: ${memory()} MB`);

let obj1 = {
  a: new Array(1024 * 1024 * 10)
};

let obj2 = {
  a: new Array(1024 * 1024 * 10)
};

console.log(`after new Array: ${memory()} MB`);

// 追踪obj1对象的GC
trackGC(obj1, () => {
  console.log("obj1 gc");
});

// 追踪obj2对象的GC
trackGC(obj2, () => {
  console.log("obj2 gc");
});

global.gc();

console.log(`after gc 1: ${memory()} MB`);

obj1 = null;
obj2 = null;

global.gc();

console.log(`after gc 2: ${memory()} MB`);
```

执行上面代码 `node --expose-gc test.js`，输出如下

```javascript
before new Array: 6 MB
after new Array: 166 MB
after gc 1: 165 MB
after gc 2: 5 MB
obj2 gc // 说明obj2被gc了
obj1 gc // 说明obj1被gc了
```

从输出中可以看到 `obj1 和 ` `obj2` 变量被 `GC` 了



**如何知道 AsyncResource 对象被 GC 了？**

当创建一个 `AsyncResource` 对象时，默认会执行 `registerDestroyHook`

```c++
// node-18.15.0/lib/async_hooks.js
class AsyncResource {
  constructor(type, opts = kEmptyObject) {
    // ...
    
    if (!requireManualDestroy && destroyHooksExist()) {
      // This prop name (destroyed) has to be synchronized with C++
      const destroyed = { destroyed: false };
      this[destroyedSymbol] = destroyed;
      
      // 执行 registerDestroyHook，用于注册一个回调函数，在异步操作执行完成后自动调用以释放资源
      registerDestroyHook(this, asyncId, destroyed);
    }
  }

}

// node-18.15.0/src/async_wrap.cc
class DestroyParam {
 public:
  double asyncId;
  Environment* env;
  Global<Object> target;
  Global<Object> propBag;
};

// node-18.15.0/src/async_wrap.cc
static void RegisterDestroyHook(const FunctionCallbackInfo<Value>& args) {
  ...
  Isolate* isolate = args.GetIsolate();
  
  DestroyParam* p = new DestroyParam();
  
  p->asyncId = args[1].As<Number>()->Value();
  p->env = Environment::GetCurrent(args);
  
  // args[0] 为 JS 层的 AsyncResource 对象
  p->target.Reset(isolate, args[0].As<Object>());
  
  ...
  // 设置弱引用回调，p 为回调时传入的参数，回调函数为 AsyncWrap::WeakCallback
  p->target.SetWeak(p, AsyncWrap::WeakCallback, WeakCallbackType::kParameter);
  ...
}
```

`RegisterDestroyHook` 中创建了一个 `DestroyParam` 对象保存上下文，然后调用 `SetWeak` 设置了 `JS` 对象的弱引用回调，当 `AsyncResource` 没有被其他变量引用时就会被垃圾回收，从而执行 `AsyncWrap::WeakCallback`，其代码如下

```c++
// node-18.15.0/src/async_wrap.cc
void AsyncWrap::WeakCallback(const WeakCallbackInfo<DestroyParam>& info) {
  HandleScope scope(info.GetIsolate());
  
  // 智能指针，执行完 WeakCallback 后释放堆对象 DestroyParam 内存
  std::unique_ptr<DestroyParam> p{info.GetParameter()};
  
  Local<Object> prop_bag = PersistentToLocal::Default(info.GetIsolate(),
                                                      p->propBag);
  Local<Value> val;
  
  p->env->RemoveCleanupHook(DestroyParamCleanupHook, p.get());

  // 触发 async_hooks 的 destroy 钩子函数
  if (val.IsEmpty() || val->IsFalse()) {
    AsyncWrap::EmitDestroy(p->env, p->asyncId);
  }
}

// node-18.15.0/src/async_wrap.cc
// destroy 钩子函数
void AsyncWrap::EmitDestroy(Environment* env, double async_id) {
  if (env->async_hooks()->fields()[AsyncHooks::kDestroy] == 0 ||
      !env->can_call_into_js()) {
    return;
  }

  if (env->destroy_async_id_list()->empty()) {
    env->SetImmediate(&DestroyAsyncIdsCallback, CallbackFlags::kUnrefed);
  }

  // If the list gets very large empty it faster using a Microtask.
  // Microtasks can't be added in GC context therefore we use an
  // interrupt to get this Microtask scheduled as fast as possible.
  if (env->destroy_async_id_list()->size() == 16384) {
    env->RequestInterrupt([](Environment* env) {
      env->context()->GetMicrotaskQueue()->EnqueueMicrotask(
        env->isolate(),
        [](void* arg) {
          DestroyAsyncIdsCallback(static_cast<Environment*>(arg));
        }, env);
      });
  }

  env->destroy_async_id_list()->push_back(async_id);
}
```

`AsyncWrap::WeakCallback` 中通过 `AsyncWrap::EmitDestroy` 触发了 `async_hooks` 的 `destroy` 钩子，从而通过 `destroy` 钩子的 `asyncId` 就可以知道哪个 `AsyncResource` 对象被 `GC` 了，从而根据 `WeakMap` 的映射关系知道哪个被追踪的 `JS` 对象被 `GC` 了



# 7 总结

本文介绍了 `Node.js` 中 `BaseObject` 的 `MakeWeak()` 弱引用机制，它是内存管理机制的基础；也介绍了下面 4 种内存管理机制

1. `trace_events` 模块：关联了 `C++` 的 `JS` 层对象，是通过弱引用机制进行 `JS` 对象和 `C++` 对象的内存管理

2. `diagnostics_channel` 模块：基于引用计数进行内存管理，底层也有使用弱引用机制

3. `HandleWrap` 类：是对 `libuv handle` 的封装，所以不再使用时需要显式调用 `close` 关闭 `handle`，才能释放内存

4. `ReqWrap` 类： 是对请求的封装，是一次性的操作，从发起连接到结束连接整个过程都是由 `Node.js` 自动管理内存

   

> 本文是学习 [theanarkh](https://github.com/theanarkh) 大佬的 [《Node.js 中 JS 和 C++ 对象的内存管理机制》](https://juejin.cn/book/7171733571638738952/section/7174421912490082341?enter_from=course_center&utm_source=course_center) 后根据自己的理解所整理的，若有不理解的地方，建议阅读原文



# 8 参考

* [Node.js v18.15.0](https://github.com/nodejs/node/tree/v18.15.0)

* [Node.js 中 JS 和 C++ 对象的内存管理机制](https://juejin.cn/book/7171733571638738952/section/7174421912490082341?enter_from=course_center&utm_source=course_center)

* [如何追踪 JS 对象是否被 GC](https://zhuanlan.zhihu.com/p/551005752)

