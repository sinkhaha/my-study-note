> 本文 `Node.js` 的版本 v18.15.0 、`libuv` 的版本 `v1.44.2` （`libuv` 相关只涉及 `unix` 平台）
>
> 前置知识：需要先了解 `V8` 相关基础，如实例、上下文、句柄、函数模板等

# 1 Node.js 中 C++ 层的核心类

因为 `Node.js` 底层用到了很多 `C / C++` 的库，所以在 `Node.js` 中实现了一层 `C++` 层，它负责 `JS` 层跟底层 `C / C++` 库的交互，`Node.js` 中`C++` 层向上会暴露接口给 `JS` 层使用，`C++` 层向下会跟 `libuv` 通信

> 以调用 `libuv` 为例，当我们调用 `JS` 层代码时，`JS` 层会先调用 `C++` 层暴露的接口， `C++` 层继续调用 `libuv` 的接口，`libuv` 完成操作后，会回调 `C++` 层，最终 `C++` 层 再回调 `JS` 层



`Node.js` 在 `C++` 层设计了一些核心类，`JS` 层跟 `C++` 层、`C++` 层 跟 `libuv` 的通信都依赖于这些类，如

* `BaseObject`
* `AsyncWrap`
* `HandleWrap`
* `ReqWrap`

它们的关系如下图



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/%E7%B1%BB%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.drawio.png)



## 1.1 BaseObject 基类

`BaseObject` 类是 `Node.js` 中所有自定义 `JS` 对象的基类，用于管理 `JS` 对象和所关联 `C++` 对象的生命周期。



`Node.js` 的 [BaseObject](https://github.com/nodejs/node/blob/v18.15.0/src/base_object.h#L45) 类定义在 `base_object.h` 文件中。

> `Node.js` 中还存在 `base_object-inl.h` 文件，这个文件是 `base_object.h` 的内联（`inline`）实现文件，它实现了一些 `BaseObject` 类的内联方法和属性。内联方法和属性通常是一些简单的、频繁调用的方法或属性，使用内联方法和属性可以提高代码的执行效率



### 1.1.1 BaseObject 的定义和实现

**BaseObject 类的定义如下**

```c++
// node-18.15.0/src/base_object.h
class BaseObject : public MemoryRetainer {  
  public:
    ...
    // 构造函数的定义
    BaseObject(Realm* realm, v8::Local<v8::Object> object);
    inline BaseObject(Environment* env, v8::Local<v8::Object> object);
  
    static inline void SetInternalFields(v8::Local<v8::Object> object,
                                       void* slot);
    ...
      
 private:  
  // persistent_handle_ 持久句柄，用于保存与 BaseObject 相关联的 JS 对象的句柄，并且通过弱引用机制管理 JS 和 C++ 对象的生命周期
  v8::Global<v8::Object> persistent_handle_; 
  ...
};  
```



**BaseObject 类的构造函数实现如下**

```c++
// node-18.15.0/src/base_object-inl.h
// 构造函数的实现，这里的 BaseObject(env->principal_realm(), object) 利用构造函数初始化列表语法初始化
BaseObject::BaseObject(Environment* env, v8::Local<v8::Object> object)
    : BaseObject(env->principal_realm(), object) {}

// node-18.15.0/src/base_object.cc
// 构造函数的实现
// 这里用到了构造函数初始化列表语法，去初始化成员变量 persistent_handle_ 和 realm_
// （1）persistent_handle_ 被初始化为一个 Persistent 句柄，这里可理解为 传入realm->isolate() 和 object 调用 v8::Persistent 类的构造函数去初始化 persistent_handle_（realm->isolate() 表示与 BaseObject 相关联的 JS 对象所属的 V8 Isolate，而 object 表示要持久化保存的 JS 对象）
// （2）realm_ 是一个指向 Realm 对象的指针，表示与 BaseObject 相关联的执行环境。这里可理解为 将 realm_ 成员变量设置为传递给构造函数的 realm 参数

// 重点
// 这里 persistent_handle_的作用：persistent_handle_用于保存与 BaseObject 相关联的 JS 对象，需要使用时可以通过 object() 函数取出该 js 对象
BaseObject::BaseObject(Realm* realm, Local<Object> object)
    : persistent_handle_(realm->isolate(), object), realm_(realm) {
  CHECK_EQ(false, object.IsEmpty());
  CHECK_GE(object->InternalFieldCount(), BaseObject::kInternalFieldCount);
      
  // 调用 SetInternalFields()   
  SetInternalFields(object, static_cast<void*>(this));
      
  ...
}

// node-18.15.0/src/base_object-inl.h
// 重点：SetInternalFields的作用：设置对象的内部字段，即把 this (当前的BaseObject对象) 存到 js 对象(object)中   
void BaseObject::SetInternalFields(v8::Local<v8::Object> object, void* slot) {
  TagNodeObject(object);
  // 把 this (当前的BaseObject对象) 存到 js 对象(object)中    
  // node-18.15.0 中 BaseObject::kSlot 枚举值为1
  object->SetAlignedPointerInInternalField(BaseObject::kSlot, slot);
}
```

由代码可知，`BaseObject` 类在构造函数中实现了 `JS` 对象和 `C++` 对象（如 `BaseObject`）的互相关联；其本质是 `C++` 对象使用了 `persistent_handle_` 变量存储了 `JS` 对象，`JS` 对象通过 `SetAlignedPointerInInternalField` 使用了 `V8` 的内置字段存储了 `C++` 对象。其关联关系如下图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/BaseObject%E5%AF%B9%E8%B1%A1.png)

### 1.1.2 获取 JS 对象：object()

`BaseObject` 类的 [`object()`](https://github.com/nodejs/node/blob/v18.15.0/src/base_object-inl.h#L54)  函数的作用是从 `C++` 对象中获取 `JS` 对象，`object()` 的定义和实现如下

```c++
// object() 的定义
// node-18.15.10/src/base_object.h
class BaseObject : public MemoryRetainer {
 public:
   // 返回 JS 对象
   inline v8::Local<v8::Object> object() const;
 ...
}

// object() 的实现
// node-18.15.10/src/base_object-inl.h
v8::Local<v8::Object> BaseObject::object() const {
  // 返回 js 对象
  return PersistentToLocal::Default(env()->isolate(), persistent_handle_);
}
```

由代码可知，`object()` 本质是从 `BaseObject` 类的 `persistent_handle_` 变量中获取 `JS` 对象



### 1.1.3 获取 C++ 对象：FromJSObject()

`BaseObject` 类的 [`FromJSObject()`](https://github.com/nodejs/node/blob/v18.15.0/src/base_object-inl.h#L87) 函数的作用是从 `JS` 对象中 获取 `C++` 对象（如 `BaseObject` 对象），它是一个静态函数，有两个定义

* `static inline BaseObject* FromJSObject(v8::Local<v8::Value> object)`： 返回 `BaseObject` 指针，这个方法只能将 `JS` 对象转换为 `BaseObject` 类型的对象
* `static inline T* FromJSObject(v8::Local<v8::Value> object)`：是一个模板函数，返回指定类型（`BaseObject` 及其子类）的指针，这个方法会将传入的 `JS` 对象转换为指定的 `C++` 类型



**FromJSObject 的定义和实现如下**

```c++
// 定义
// node-18.15.10/src/base_obejct.h
class BaseObject : public MemoryRetainer {
  public:
    // 定义1:返回 BaseObject 指针
    static inline BaseObject* FromJSObject(v8::Local<v8::Value> object);
    // 定义2:返回指定类型(BaseObject的子类)的指针
    static inline T* FromJSObject(v8::Local<v8::Value> object);
  ...
}

// 实现分别如下
// node-18.15.10/src/base_object-inl.h
// 从 js对象(obj)中 取出里面保存的 BaseObject 对象  
BaseObject* BaseObject::FromJSObject(v8::Local<v8::Value> value) {
  v8::Local<v8::Object> obj = value.As<v8::Object>();
  DCHECK_GE(obj->InternalFieldCount(), BaseObject::kInternalFieldCount);
  
  // 从 js 对象中 取出 BaseObject 对象（static_cast<BaseObject*>(xx)强制转成 BaseObject* 类型）
  return static_cast<BaseObject*>(
      obj->GetAlignedPointerFromInternalField(BaseObject::kSlot));
}

// T 为 BaseObject 子类
// ASSIGN_OR_RETURN_UNWRAP 宏调用了这个函数，BaseObject 的 Unwrap() 函数也调用了这个函数
template <typename T>
T* BaseObject::FromJSObject(v8::Local<v8::Value> object) {
  // static_cast<T*>(xx) 强制把 BaseObject* 转成 T* 类型
  return static_cast<T*>(FromJSObject(object));
}
```

由代码可知，`FromJSObject()` 函数本质是通过 `GetAlignedPointerFromInternalField` 从 `JS` 对象的内置字段中获取 `C++` 对象。



在 `Node.js` 中，基本每次从 `JS` 层调用 `C++` 层的函数时都用到了 `FromJSObject()` 函数，例如在 `ASSIGN_OR_RETURN_UNWRAP` 宏中就调用了 `FromJSObject()` 函数



## 1.2 AsyncWrap 类

[`AsyncWrap`](https://github.com/nodejs/node/blob/v18.15.0/src/async_wrap.h#L115) 类定义在 `async_wrap.h` 中，`AsyncWrap` 继承 `BaseObject`，其作用如下

1. 管理异步操作的生命周期 ：它可以跟踪异步操作的状态，并在异步操作完成时通知 `Node.js` 事件循环；还可以在异步操作发生错误时，记录错误信息并返回给 `JS` 层
2. 管理异步操作的资源：它可以管理异步操作使用的资源，包括内存、文件句柄、网络连接等；也可以在异步操作完成时，释放这些资源，以避免内存泄漏等问题
3. 提供了一些 `API` 用于在异步操作的不同阶段执行回调函数：例如可以在异步操作开始时执行一个回调函数，在异步操作完成时执行另一个回调函数

> `async_hook` 模块就是基于此 `AsyncWrap` 类实现的



### 1.2.1 MakeCallback 函数的定义和实现

`AsyncWrap` 类在异步操作中， `C++` 层会回调 `JS` 层，其核心逻辑主要是 [`MakeCallback()`](https://github.com/nodejs/node/blob/v18.15.0/src/async_wrap.h#L196) 函数。



**AsyncWrap 类中 MakeCallback 函数的定义如下**

```c++
// node-18.15.0/src/async_wrap.h
// AsyncWrap 有多 个MakeCallback声明，我们只看下面这个 v8::Name 类型的 symbol 
class AsyncWrap : public BaseObject {
 public:
  // MakeCallback 定义
  inline v8::MaybeLocal<v8::Value> MakeCallback(
      const v8::Local<v8::Name> symbol, // symbol 是一个表示要调用的函数的名称或符号 的 Name 对象
      int argc,
      v8::Local<v8::Value>* argv);
  ...
}

// MakeCallback 的实现
// node-18.15.0/src/async_wrap-inl.h
inline v8::MaybeLocal<v8::Value> AsyncWrap::MakeCallback(
    const v8::Local<v8::Name> symbol,
    int argc,
    v8::Local<v8::Value>* argv) {
  
  v8::Local<v8::Value> cb_v;
  
  // 从 js 对象中取出该 symbol 属性对应的值，值是个函数，赋予cb_v
  // symbol 的值通常在 js 层设置，比如 onread = xxx，oncomplete = xxx  
  if (!object()->Get(env()->context(), symbol).ToLocal(&cb_v))
    return v8::MaybeLocal<v8::Value>();
  
  // 判断是否是函数
  if (!cb_v->IsFunction()) {
    v8::Isolate* isolate = env()->isolate();
    return Undefined(isolate);
  }
  
   // 调用MakeCallback执行该函数 cb_v ，具体实现见下方
  return MakeCallback(cb_v.As<v8::Function>(), argc, argv);
}
```



**AsyncWrap 中 MakeCallback 函数的实现如下**

```c++
// 定义 node-18.15.0/src/async_wrap.h
v8::MaybeLocal<v8::Value> MakeCallback(const v8::Local<v8::Function> cb,
                                         int argc,
                                         v8::Local<v8::Value>* argv);

// 实现 node-18.15.0/src/async_wrap.cc
MaybeLocal<Value> AsyncWrap::MakeCallback(const Local<Function> cb,
                                          int argc,
                                          Local<Value>* argv) {
  EmitTraceEventBefore();

  ProviderType provider = provider_type();
  async_context context { get_async_id(), get_trigger_async_id() };
  
  // 重点：调用了 InternalMakeCallback 函数，进而执行 js 层回调
  MaybeLocal<Value> ret = InternalMakeCallback(
      env(), object(), object(), cb, argc, argv, context);

  // This is a static call with cached values because the `this` object may
  // no longer be alive at this point.
  EmitTraceEventAfter(provider, context.async_id);

  return ret;
}

// InternalMakeCallback 函数的实现
// node-18.15.0/src/api/callback.cc
MaybeLocal<Value> InternalMakeCallback(Environment* env,
                                       Local<Object> resource,
                                       Local<Object> recv,
                                       const Local<Function> callback,
                                       int argc,
                                       Local<Value> argv[],
                                       async_context asyncContext) { 
  ......
  // 执行 JS 层回调；即通过 V8 Function 的 Call 执行该 JS 函数
  ret = callback->Call(context, recv, argc, argv);
  ......
} 
```

由代码可知，`MakeCallback` 函数最终调用的是 `InternalMakeCallback` 函数，进而执行 `JS`  回调函数



## 1.3 HandleWrap 类

[HandleWrap](https://github.com/nodejs/node/blob/v18.15.0/src/handle_wrap.h#L57) 类定义在 `handle_wrap.h` 中，`HandleWrap` 类继承 `AsyncWrap` 类，`HandleWrap` 类是对 `libuv uv_handle_t` 结构体和操作的封装，也是很多 `C++` 类的基类（比如 `TCP / UDP`）

* `HandleWrap` 有个 `uv_handle_t*` 类型的 `handle_` 变量，它指向其子类的 `handle` 结构体，比如 `TCPWrap` 类是`HandleWrap` 的子类（此时是 `uv_tcp_t` 类型，`TCPWrap` 后面会讲解）
* `HandleWrap` 还有对 `handle` 管理的功能，比如 `Ref` 和 `Unref` 用于控制该 `handle` 是否影响事件循环的退出



### 1.3.1 HandleWrap 类定义和实现

**HandleWrap 类的定义如下**

```c++
// 定义 node-18.15.0/src/handle_wrap.h
class HandleWrap : public AsyncWrap {  
  // 一些操作和判断 handle 状态函数的定义，实际是对 libuv 函数的封装  
 public:
  static void Close(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void Ref(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void Unref(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void HasRef(const v8::FunctionCallbackInfo<v8::Value>& args);

  static inline bool IsAlive(const HandleWrap* wrap) {
    return wrap != nullptr &&
        wrap->IsDoneInitializing() &&
        wrap->state_ != kClosed;
  }
  
  static inline bool HasRef(const HandleWrap* wrap) {
    return IsAlive(wrap) && uv_has_ref(wrap->GetHandle());
  }

  // 获取封装的 libuv 的 uv_handle_t
  inline uv_handle_t* GetHandle() const { return handle_; }

  // 关闭 handle，如果传入了回调，该回调会在 close 阶段被执行  
  virtual void Close(
      v8::Local<v8::Value> close_callback = v8::Local<v8::Value>());

  static v8::Local<v8::FunctionTemplate> GetConstructorTemplate(
      Environment* env);
  static void RegisterExternalReferences(ExternalReferenceRegistry* registry);

  
 protected:
  HandleWrap(Environment* env,
             v8::Local<v8::Object> object,
             uv_handle_t* handle,
             AsyncWrap::ProviderType provider);
  virtual void OnClose() {}  // 子类可实现
  void OnGCCollect() final;
  bool IsNotIndicativeOfMemoryLeakAtExit() const override;

  void MarkAsInitialized();
  void MarkAsUninitialized();
  
  // handle 的状态 
  inline bool IsHandleClosing() const {
    return state_ == kClosing || state_ == kClosed;
  }
  
  static void OnClose(uv_handle_t* handle);
  enum { kInitialized, kClosing, kClosed } state_; // handle 的状态 

private:
  friend class Environment;
  friend void GetActiveHandles(const v8::FunctionCallbackInfo<v8::Value>&);

  friend int GenDebugSymbols();
  ListNode<HandleWrap> handle_wrap_queue_;  // handle 队列  
  
  // 保存 handle
  // uv_handle_t 类型是所有 handle 的基类，它指向其子类的 handle 类结构体，比如 TCPWrap 类是HandleWrap 的子类（此时是uv_tcp_t 类型）
  uv_handle_t* const handle_;  
  
};  
```



**HandleWrap 类构造函数的实现如下**

```c++
// HandleWrap 构造函数的实现 node-18.15.0/src/handle_wrap.cc
HandleWrap::HandleWrap(Environment* env,
                       Local<Object> object, // object 为 JS 层对象 
                       uv_handle_t* handle, // handle 为子类具体的 handle 类型，不同模块不一样 
                       AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      state_(kInitialized),
      handle_(handle) { // 实例化handle_
        
  // 保存 libuv handle 和 C++ 对象的关系，libuv 执行 C++ 回调时使用
  handle_->data = this;
        
  HandleScope scope(env->isolate());
  CHECK(env->has_run_bootstrapping_code());
  env->handle_wrap_queue()->PushBack(this);
}
```

由代码可知，`HandleWrap` 构造函数的主要逻辑是 用 `handle_`  变量保存了 `libuv` 的句柄，`handle_` 句柄有个 `data` 变量，这个 `data` 又保存了 `HandleWrap` 类型的对象，所以在 `libuv` 中可以通过这个 `handle_` 的 `data` 得到它所关联的 `C++` 对象，进而在 `libuv` 层操作 `C++` 层



**HandleWrap 类的结构图如下**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/HandleWrap%E7%B1%BB1.png)



**AsyncWrap 其他函数的定义和实现**

再看一下 `HandleWrap` 类判断和操作 `handle` 状态的函数的实现

```c++
// 实现 node-18.15.0/src/handle_wrap.cc
// 修改 handle 为活跃状态  
void HandleWrap::Ref(const FunctionCallbackInfo<Value>& args) {
  HandleWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

  if (IsAlive(wrap))
    uv_ref(wrap->GetHandle());
}

// 修改 hande 为不活跃状态  
void HandleWrap::Unref(const FunctionCallbackInfo<Value>& args) {
  HandleWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

  if (IsAlive(wrap))
    uv_unref(wrap->GetHandle());
}

// 判断 handle 是否处于活跃状态  
void HandleWrap::HasRef(const FunctionCallbackInfo<Value>& args) {
  HandleWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());
  args.GetReturnValue().Set(HasRef(wrap));
}

// 关闭 handle（JS 层调用） 
void HandleWrap::Close(const FunctionCallbackInfo<Value>& args) {
  HandleWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

  // 调用真正的Close函数 void HandleWrap::Close(Local<Value> close_callback)
  wrap->Close(args[0]);
}

// 真正关闭 handle 的函数  
void HandleWrap::Close(Local<Value> close_callback) {
  // 正在关闭或已经关闭  
  if (state_ != kInitialized)
    return;

  // 调用 libuv 函数，OnClose是libuv执行完后的回调函数
  uv_close(handle_, OnClose);
  // 关闭中 
  state_ = kClosing;

  // 传了 onclose 回调则保存起来，在 close 阶段后调用 
  if (!close_callback.IsEmpty() && close_callback->IsFunction() &&
      !persistent().IsEmpty()) {
    object()->Set(env()->context(),
                  env()->handle_onclose_symbol(),
                  close_callback).Check();
  }
}

// 关闭 handle 成功后回调，libuv 层执行  
void HandleWrap::OnClose(uv_handle_t* handle) {
  CHECK_NOT_NULL(handle->data);
  BaseObjectPtr<HandleWrap> wrap { static_cast<HandleWrap*>(handle->data) };
  wrap->Detach();

  Environment* env = wrap->env();
  HandleScope scope(env->isolate());
  Context::Scope context_scope(env->context());

  CHECK_EQ(wrap->state_, kClosing);

  wrap->state_ = kClosed;

  // 执行子类的 onClose，如果没有则是空操作
  wrap->OnClose();
  wrap->handle_wrap_queue_.Remove();

  // 有 JS 层 onclose 回调则执行  
  if (!wrap->persistent().IsEmpty() &&
      wrap->object()->Has(env->context(), env->handle_onclose_symbol())
      .FromMaybe(false)) {
    wrap->MakeCallback(env->handle_onclose_symbol(), 0, nullptr);
  }
}
```



## 1.4 ReqWrap 类

[ReqWrap](https://github.com/nodejs/node/blob/v18.15.0/src/req_wrap.h#L31) 类定义在 `req_wrap.h` 中， `ReqWrap` 继承自 `AsyncWrap`，它是请求操作的基类，可以有不同的子类实现（比如 `ConnectWrap` 类就实现了 `ReqWrap`）。



`ReqWrap` 抽象了请求 `libuv` 的过程，实际具体数据结构和操作由其子类实现。看一下 `ConnectWrap` 子类的实现

```c++
// node-18.15.0/src/connect_wrap.h
// 请求 Libuv 时，数据结构是 uv_connect_t，表示一次连接请求  
class ConnectWrap : public ReqWrap<uv_connect_t> {
 public:
  ConnectWrap(Environment* env,
              v8::Local<v8::Object> req_wrap_obj,
              AsyncWrap::ProviderType provider);

  ...
};
```

接着看下建立连接时 `TCPWrap` 的 `Connect` 函数，如下

```c++
// node-18.15.0/src/tcp_wrap.cc 
void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args,
    std::function<int(const char* ip_address, T* addr)> uv_ip_addr) {
  ...
  // req_wrap_obj JS 层传来的 req 对象
  ConnectWrap* req_wrap =
        new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);

  // 发起请求，调用了Dispatch 函数，AfterConnect为回调函数
  err = req_wrap->Dispatch(uv_tcp_connect,
                          &wrap->handle_,
                          reinterpret_cast<const sockaddr*>(&addr),
                          AfterConnect); // 回调
  ...
}
```

由代码可知，`Connect` 实际是通过 `reqWrap` 类的 `Dispath` 函数调用 `libuv` 的函数



### 1.4.1 ReqWrap 类的定义和实现 

**ReqWrap 类的定义如下**

```c++
// node-18.15.0/src/req_wrap.h
template <typename T>  
class ReqWrap : public AsyncWrap, public ReqWrapBase { 
 ... 
   
 public:
    // 获取 libuv 请求结构体
    T* req() { return &req_; } 
  
 ...
   
 protected:  
  // libuv 请求结构体，类型由子类决定，如 ConnectWrap子类，此时就是 uv_connect_t 结构体
  T req_;  
  ...
};   

// node-18.15.0/src/upd_wrap.cc
// ConnectWrap 实现了 ReqWrap，此时 ReqWrap 里的 libuv 结构体类型就是 uv_connect_t
class SendWrap : public ReqWrap<uv_udp_send_t> {
...
}
```

**ReqWrap 类构造函数的实现如下**

```c++
// node-18.15.0/src/req_wrap-inl.h

// 构造函数的实现
template <typename T>
ReqWrap<T>::ReqWrap(Environment* env,
                    v8::Local<v8::Object> object,
                    AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      ReqWrapBase(env) {
  MakeWeak(); // 设置为弱持久句柄，实现在BaseObject类中实现，用于回收C++对象
  Reset(); // 初始化状态 
}

// node-18.15.0/src/req_wrap-inl.h
// Reset()的实现，作用是进行重置字段
template <typename T>
void ReqWrap<T>::Reset() {
  // 由 libuv 调用的 C++ 层回调
  original_callback_ = nullptr;
  req_.data = nullptr;
}
```



### 1.4.2 Dispatch 的定义和实现

`Dispatch` 函数的作用是在 `ReqWrap` 类中调用 `libuv` 的函数

> 当 `Node.js` 的 `tcp` 通过 `connect()` 发起一个请求时，会先创建一个 `ReqWrap` 的子类对象`ConnectWrap` ，接着调用 `Dispatch` 发起真正的请求；`Dispatch` 实际是通过 `CallLibuvFunction<T, LibuvFunction>::Call` 调用 `libuv` 的函数



**Dispatch 函数的定义和实现如下**

```c++
// node-18.15.0/src/req_wrap.h
// req() 用于获取 libuv 请求结构体
template <typename T>
class ReqWrap : public AsyncWrap, public ReqWrapBase {
  T* req() { return &req_; }  
  ...
}

// node-18.15.0/src/req_wrap-inl.h
// Dispatched() 用于保存 libuv 结构体和 ReqWrap 实例的关系，发起请求时调用  
template <typename T>
void ReqWrap<T>::Dispatched() {
  req_.data = this; // libuv 的结构体保存了当前的 C++对象
}

// node-18.15.0/src/req_wrap-inl.h
// 该函数作用是：在 ReqWrap 中调用 libuv 函数  
template <typename T>
template <typename LibuvFunction, typename... Args>
int ReqWrap<T>::Dispatch(LibuvFunction fn, Args... args) {
  // 关联 libuv 结构体和 C++ 请求对象的关系（比如 ConnectWrap 类对象）
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
  
  // 调用 CallLibuvFunction 执行 libuv 的函数
  int err = CallLibuvFunction<T, LibuvFunction>::Call(
      fn, // libuv 函数
      env()->event_loop(),
      req(),
      // libuv 执行完成后会执行该回调函数
      MakeLibuvRequestCallback<T, Args>::For(this, args)...);
  
  if (err >= 0) {
    ClearWeak();
    env()->IncreaseWaitingRequestCounter();
  }
  
  return err;
}
```

分析如下

* 先调用 `Dispatched()` 关联 `libuv` 结构体和 `C++` 请求对象的关系（比如 `ConnectWrap` 类对象）
* 通过 `CallLibuvFunction` 函数调用 `libuv` 的函数
* 通过 `MakeLibuvRequestCallback::For` 函数设置 `libuv` 的回调函数，`libuv` 执行完成后会执行该回调函数



**CallLibuvFunction 函数**

`CallLibuvFunction` 作用是调用 `libuv` 的函数，`Node.js` 针对不了的 `libuv` 函数签名格式编写了不同的 `CallLibuvFunction` 模板函数。定义如下

```c++
// CallLibuvFunction的模板函数如下

// node-18.15.0/src/req_wrap-inl.h

// Invoke a generic libuv function that initializes uv_req_t instances.
// This is, unfortunately, necessary since they come in three different
// variants that can not all be invoked in the same way:
// - int uv_foo(uv_loop_t* loop, uv_req_t* request, ...);
// - int uv_foo(uv_req_t* request, ...);
// - void uv_foo(uv_req_t* request, ...);template <typename ReqT, typename T>
struct CallLibuvFunction;

// Detect `int uv_foo(uv_loop_t* loop, uv_req_t* request, ...);`.
template <typename ReqT, typename... Args>
struct CallLibuvFunction<ReqT, int(*)(uv_loop_t*, ReqT*, Args...)> {
  using T = int(*)(uv_loop_t*, ReqT*, Args...);
  template <typename... PassedArgs>
  // 执行并返回 fn 函数
  static int Call(T fn, uv_loop_t* loop, ReqT* req, PassedArgs... args) {
    return fn(loop, req, args...);
  }
};

// Detect `int uv_foo(uv_req_t* request, ...);`.
template <typename ReqT, typename... Args>
struct CallLibuvFunction<ReqT, int(*)(ReqT*, Args...)> {
  using T = int(*)(ReqT*, Args...);
  template <typename... PassedArgs>
  static int Call(T fn, uv_loop_t* loop, ReqT* req, PassedArgs... args) {
    return fn(req, args...);
  }
};

// Detect `void uv_foo(uv_req_t* request, ...);`.
template <typename ReqT, typename... Args>
struct CallLibuvFunction<ReqT, void(*)(ReqT*, Args...)> {
  using T = void(*)(ReqT*, Args...);
  template <typename... PassedArgs>
  static int Call(T fn, uv_loop_t* loop, ReqT* req, PassedArgs... args) {
    fn(req, args...);
    return 0;
  }
};
```



**MakeLibuvRequestCallback 函数**

`MakeLibuvRequestCallback` 函数的作用设置 `libuv` 的回调函数，`libuv` 异步操作执行完成后会执行该回调函数，定义如下

```c++
// node-18.15.0/src/req_wrap-inl.h

// from_req 函数通过 req 成员找所属对象的地址
template <typename T>
ReqWrap<T>* ReqWrap<T>::from_req(T* req) {
  return ContainerOf(&ReqWrap<T>::req_, req);
}

// 模板函数1，这个模板匹配第2个参数为非函数的参数
template <typename ReqT, typename T>
struct MakeLibuvRequestCallback {
  static T For(ReqWrap<ReqT>* req_wrap, T v) {
    static_assert(!is_callable<T>::value,
                  "MakeLibuvRequestCallback missed a callback");
    return v;
  }
};

// 模板函数2，这个模板匹配第2个参数为函数的参数
// Match the `void callback(uv_req_t*, ...);` signature that all libuv
// callbacks use.
template <typename ReqT, typename... Args>
struct MakeLibuvRequestCallback<ReqT, void(*)(ReqT*, Args...)> {
  using F = void(*)(ReqT* req, Args... args);

  // libuv 回调
  static void Wrapper(ReqT* req, Args... args) {
    // 调用 from_req()，通过 libuv 结构体拿到对应的 C++ 对象
    BaseObjectPtr<ReqWrap<ReqT>> req_wrap{ReqWrap<ReqT>::from_req(req)};
    req_wrap->Detach();
    req_wrap->env()->DecreaseWaitingRequestCounter();
    
    // 拿到原始的回调函数并执行
    F original_callback = reinterpret_cast<F>(req_wrap->original_callback_);
    original_callback(req, args...);
  }

  // For 的第二个参数为函数
  static F For(ReqWrap<ReqT>* req_wrap, F v) {
    CHECK_NULL(req_wrap->original_callback_);
    
    // 保存原来的函数
    req_wrap->original_callback_ =
        reinterpret_cast<typename ReqWrap<ReqT>::callback_t>(v);
    
    // 返回包裹函数，这个包裹的 Wrapper 函数会被传入 libuv，等到 libuv 回调时，Wrapper 里再执行真正的回调
    return Wrapper;
  }
};
```



`MakeLibuvRequestCallback::For` 可用于适配不同的 `Dispatch` 调用格式（ `TCP` 连接和 `DNS` 解析），例如

```c++
// tcp_wrap.cc 
// TCP 连接
 err = req_wrap->Dispatch(uv_tcp_connect,
                          &wrap->handle_,
                          reinterpret_cast<const sockaddr*>(&addr),
                          AfterConnect); // 回调

// cares_wrap.cc
// DNS 解析
int err = req_wrap->Dispatch(uv_getaddrinfo,
                             AfterGetAddrInfo, // 回调
                             *hostname,
                             nullptr,
                             &hints);
```



**执行 `Dispatch` 后的结构图如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/ReqWrap1.png)



# 2 Node.js 和 C++ / libuv 的交互

下面以 `tcp` 连接为例，来了解 `Node.js` 中 `JS` 层、`C++` 层、`libuv` 层之间的交互



## 2.1 JS 调用 C++

`JS` 层能调用 `C++` 层的函数，主要是利用了 `V8` 可扩展的能力，类似 `JS` 调用 `C++` 扩展模块。

在 `Node.js` 中，内置的 `C++` 模块定义了一系列函数（比如 `tcp_wrap.cc`），然后会在 `C++` 层创建一个 `JS` 函数模板对象，把 `C++` 模块的内部函数挂载到这个 `JS` 函数模板中，最后暴露这个函数模板的函数给 `JS` 层调用



### 2.1.1 new TCP()

如下 `JS` 代码

```js
// node-18.15.0/lib/net.js
const {
  TCP, // tcp_wrap.cc中的 TCP 函数
  TCPConnectWrap,
  constants: TCPConstants
} = internalBinding('tcp_wrap');

// 实际调用的是 C++ 层的 new TCP()
new TCP(...);   
```

分析如下

* 当 `JS` 层执行  `new TCP()`  时，实际调用了 `C++` 层 `TCPWrap` 类（`tcp_wrap.cc`）里的 `New` 函数

* `JS` 在调用 `C++` 模块时，需要先将 `C++` 模块（`tcp_wrap`）注册到 `Node.js` 中，这样 `Node.js` 内部就能直接通过 `internalBinding()` 加载这个 `tcp_wrap` 模块
* `Node.js` 在启动时会通过 `NODE_BINDING_CONTEXT_AWARE_INTERNAL` 宏注册 `C++` 层的 `tcp_wrap` 内置模块， `TCPWrap` 模块的初始化函数是  `Initialize` ，此时 `Node.js` 会调用这个初始化函数； `TCPWrap::Initialize`  函数的主要逻辑是创建一个 `JS` 对象，并暴露 `C++` 层 `tcp_wrap` 里的方法给 `JS` 层，这样 `JS` 层就能使用 `tcp_wrap` 里的函数了



### 2.1.2 TCPWrap 类

`TCPWrap` 类继承了 `HandleWrap`，而 `HandleWrap` 继承了 `BaseObject`。`TCPWrap` 的定义和实现如下

```c++
// TCPWrap 继承 ConnectionWrap，ConnectionWrap 继承 LibuvStreamWrap，LibuvStreamWrap 继承 HandleWrap，所以 TCPWrap 最终继承了 HandleWrap
class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t> { ... }
class ConnectionWrap : public LibuvStreamWrap { ... }
class LibuvStreamWrap : public HandleWrap, public StreamBase { ... }

// node-18.15.0/src/handle_wrap.cc
HandleWrap::HandleWrap(Environment* env,
                       Local<Object> object,
                       uv_handle_t* handle,
                       AsyncWrap::ProviderType provider)
    : AsyncWrap(env, object, provider),
      state_(kInitialized),
      handle_(handle) {
        
  // 保存 libuv handle 和 C++ 对象的关系        
  handle_->data = this;
        
  HandleScope scope(env->isolate());
  CHECK(env->has_run_bootstrapping_code());
  env->handle_wrap_queue()->PushBack(this);
}
```

因为 `HandleWrap` 在实例化时就 通过 `handle_` 变量保存了 `libuv handle`（此时 `TCPWrap` 类传的是 `uv_tcp_t`）；且 `handle_` 的 `data` 变量则保存了 `C++` 对象（此时是子类 `TCPWrap` 对象），这样 `libuv` 在执行回调时就能知道 `handle` 对应的 `C++` 对象（`TCPWrap`）



#### 2.1.2.1 TCPWrap::Initialize

再看一下 `TCPWrap` 初始化的逻辑，`TCPWrap::Initialize` 函数的实现如下

```c++
 // node-18.15.0/src/tcp_wrap.cc
void TCPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context,
                         void* priv) {
  Environment* env = Environment::GetCurrent(context);
  Isolate* isolate = env->isolate();

  // 创建一个函数模版，对应 TCPWrap::New() ，TCPWrap::New可以拿到当前this对象
  Local<FunctionTemplate> t = NewFunctionTemplate(isolate, New);
  // 内置字段，可关联的 C++ 对象个数
  t->InstanceTemplate()->SetInternalFieldCount(StreamBase::kInternalFieldCount);

  // Init properties
  t->InstanceTemplate()->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "reading"),
                             Boolean::New(env->isolate(), false));
  t->InstanceTemplate()->Set(env->owner_symbol(), Null(env->isolate()));
  t->InstanceTemplate()->Set(env->onconnection_string(), Null(env->isolate()));

  // 继承
  t->Inherit(LibuvStreamWrap::GetConstructorTemplate(env));

  // 设置原型方法，暴露C++层的方法最为同名的js层方法
  SetProtoMethod(isolate, t, "open", Open);
  SetProtoMethod(isolate, t, "bind", Bind);
  SetProtoMethod(isolate, t, "listen", Listen);
  SetProtoMethod(isolate, t, "connect", Connect);
  SetProtoMethod(isolate, t, "bind6", Bind6);
  SetProtoMethod(isolate, t, "connect6", Connect6);
  SetProtoMethod(isolate,
                 t,
                 "getsockname",
                 GetSockOrPeerName<TCPWrap, uv_tcp_getsockname>);
  SetProtoMethod(isolate,
                 t,
                 "getpeername",
                 GetSockOrPeerName<TCPWrap, uv_tcp_getpeername>);
  SetProtoMethod(isolate, t, "setNoDelay", SetNoDelay);
  SetProtoMethod(isolate, t, "setKeepAlive", SetKeepAlive);
  SetProtoMethod(isolate, t, "reset", Reset);

#ifdef _WIN32
  SetProtoMethod(isolate, t, "setSimultaneousAccepts", SetSimultaneousAccepts);
#endif

  // 设置构造方法，构造方法名字为TCP，对应的函数模板是t这个函数模板
  // SetConstructorFunction的实现在 util.cc 中
  SetConstructorFunction(context, target, "TCP", t);
  env->set_tcp_constructor_template(t);

  // Create FunctionTemplate for TCPConnectWrap.
  Local<FunctionTemplate> cwt =
      BaseObject::MakeLazilyInitializedJSTemplate(env);
  cwt->Inherit(AsyncWrap::GetConstructorTemplate(env));
  SetConstructorFunction(context, target, "TCPConnectWrap", cwt);

  // 挂载一些常量给JS层
  // Define constants
  Local<Object> constants = Object::New(env->isolate());
  NODE_DEFINE_CONSTANT(constants, SOCKET);
  NODE_DEFINE_CONSTANT(constants, SERVER);
  NODE_DEFINE_CONSTANT(constants, UV_TCP_IPV6ONLY);
  
  // 根据函数模块导出一个函数到 JS 层
  target->Set(context,
              env->constants_string(),
              constants).Check();
}
  
// 通过NODE_BINDING_CONTEXT_AWARE_INTERNAL宏注册tcp_wrap内置模块，即暴露C++层的tcp_wrap给JS层
NODE_BINDING_CONTEXT_AWARE_INTERNAL(tcp_wrap, node::TCPWrap::Initialize)
```

由代码可知，当 `JS` 层执行 `new TCP()` 时，实际会执行 `C++` 层 （`tcp_wrap.cc`）中的 `TCPWrap::New()` 函数



#### 2.1.2.2 TCPWrap::New 的实现

`TCPWrap::New` 函数的实现如下

```c++
// node-18.15.0/src/tcp_wrap.cc
void TCPWrap::New(const FunctionCallbackInfo<Value>& args) {  
  // This constructor should not be exposed to public javascript.
  // Therefore we assert that we are not trying to call this as a
  // normal function.
  CHECK(args.IsConstructCall());
  CHECK(args[0]->IsInt32());
  Environment* env = Environment::GetCurrent(args);

  int type_value = args[0].As<Int32>()->Value(); // 拿到 js 层 new TCP() 的第一个参数，即SocketType
  TCPWrap::SocketType type = static_cast<TCPWrap::SocketType>(type_value);

  ProviderType provider;
  switch (type) {
    case SOCKET:
      provider = PROVIDER_TCPWRAP;
      break;
    case SERVER:
      provider = PROVIDER_TCPSERVERWRAP;
      break;
    default:
      UNREACHABLE();
  }
    
  // args.This() 拿到this对象（即js对象）
  new TCPWrap(env, args.This(), provider);
}  

TCPWrap::TCPWrap(Environment* env, Local<Object> object, ProviderType provider)
    : ConnectionWrap(env, object, provider) {
  // 初始化libuv中的tcp句柄    
  int r = uv_tcp_init(env->event_loop(), &handle_);
  CHECK_EQ(r, 0);  // How do we proxy this error up to javascript?
                   // Suggestion: uv_tcp_init() returns void.
}
```



## 2.2 C++ 调用 libuv

### 2.2.1 Connect 建立连接

`JS` 层会调用 `tcp` 的`connect()` 函数建立连接，实际 `connect()` 会执行 `C++` 层 `tcp_wrap` 的 `Connect()` 函数

```js
// node-18.15.0/lib/net.js
function connect(...args) {
  const normalized = normalizeArgs(args);
  const options = normalized[0];
  debug('createConnection', normalized);
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

  // 调用了socket的connect，如果是tcp，即this._handle = new TCP(TCPConstants.SOCKET) 实际调用了tcp_wrap.cc 的 Connect()
  return socket.connect(normalized);
}

```



 `TCPWrap` 类的 `Connect` 函数实现如下

```c++
// node-18.15.0/src/tcp_wrap.cc
void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args) {
  CHECK(args[2]->IsUint32());
  // explicit cast to fit to libuv's type expectation
  int port = static_cast<int>(args[2].As<Uint32>()->Value());
  // 调用下面的模板函数 Connect
  Connect<sockaddr_in>(args,
                       [port](const char* ip_address, sockaddr_in* addr) {
      return uv_ip4_addr(ip_address, port, addr);
  });
}

// node-18.15.0/src/tcp_wrap.cc
template <typename T>
void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args,
    std::function<int(const char* ip_address, T* addr)> uv_ip_addr) {
  Environment* env = Environment::GetCurrent(args);

  TCPWrap* wrap; 
  
  // args.Holder() 就是取得 js 对象，比如 tcp = new TCP()
  
  // 这个宏的作用就是，从 JS 对象拿到关联的 C++ 对象（TCPWrap），然后下面就可以使用 TCPWrap 对象的 handle 去请求 libuv 了
  ASSIGN_OR_RETURN_UNWRAP(&wrap,
                          args.Holder(),
                          args.GetReturnValue().Set(UV_EBADF));
  ...
    
  // 第一个参数是 TCPConnectWrap 对象
  Local<Object> req_wrap_obj = args[0].As<Object>();
  node::Utf8Value ip_address(env->isolate(), args[1]);

  ...
  if (err == 0) {
    ...
      
    // 创建一个对象请求 libuv
    // 因为 ConnectWrap 继承了 BaseObject，req_wrap_obj 是 JS 对象，ConnectWrap 和 req_wrap_obj 会互相关联 
    ConnectWrap* req_wrap =
        new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);
   

    // 重点：这里就是 C++ 调用 libuv 的函数
    // 从 C++ 对象 TCPWrap 中获得 libuv 的 handle 结构体，调用 libuv 的 uv_tcp_connect
    err = req_wrap->Dispatch(uv_tcp_connect,
                             &wrap->handle_, // 通过 &wrap->handle_ 获取到 uv_tcp_connect 类型，因为HandleWrap类通过 handle_ 保存了libuv的句柄，TCPWrap实现了HandleWrap，此时 handle_ 是 uv_tcp_t结构体类型
                             reinterpret_cast<const sockaddr*>(&addr),
                             AfterConnect);
    
   ...
  }

  args.GetReturnValue().Set(err);
}

// ASSIGN_OR_RETURN_UNWRAP宏，里面调用了 BaseObject 类的 FromJSObject，即从 js 对象中取得 C++对象，此时是 TCPWrap
#define ASSIGN_OR_RETURN_UNWRAP(ptr, obj, ...)                                 \
  do {                                                                         \
    *ptr = static_cast<typename std::remove_reference<decltype(*ptr)>::type>(  \
        BaseObject::FromJSObject(obj));                                        \
    if (*ptr == nullptr) return __VA_ARGS__;                                   \
  } while (0)

```

由代码可知

* `C++` 层会先通过 `ASSIGN_OR_RETURN_UNWRAP` 宏调用 `FromJSObject()` 函数，通过 `FromJSObject()` 拿到 `C++` 对象（`TCPWrap`）

* 然后创建了一个 `ConnectWrap` 对象，接着通过 `Dispath` 函数调用 `libuv` 的 `uv_tcp_connect` 函数发起连接



### 2.2.2 ConnectWrap 类调用 libuv

`ConnectWrap` 继承了 `ReqWrap`类，所以也继承了 `BaseObject`，`ConnectWrap` 的定义如下

```c++
class ConnectWrap : public ReqWrap<uv_connect_t> {...}
class ReqWrap : public AsyncWrap, public ReqWrapBase { ... }
class AsyncWrap : public BaseObject {...}
```



接着看下调用 `libuv` 的过程， `req_wrap->Dispatch()` 函数会调用 `libuv`，它实际是调用了 `reqWrap` 类的 `Dispath` 函数（`Dispath` 在 `ConnectWrap` 中定义，因为 `ConnectWrap` 也继承了 `ReqWrap`）

```c++
// node-18.15.0/src/tcp_wrap.cc
// 调用 libuv 的 uv_tcp_connect() 函数发起请求
err = req_wrap->Dispatch(uv_tcp_connect,
                             &wrap->handle_,
                             reinterpret_cast<const sockaddr*>(&addr),
                             AfterConnect); // libuv连接成功后会执行的回调函数
```

 

`libuv` 的 `uv_tcp_connect` 函数如下

```c++
// node-18.15.0/deps/uv/src/uv-common.c
int uv_tcp_connect(uv_connect_t* req,  
                       uv_tcp_t* handle,  
                       const struct sockaddr* addr,  
                       uv_connect_cb cb) {  
      ...  
        
      return uv__tcp_connect(req, handle, addr, addrlen, cb);  
}  
      
// unix平台
int uv__tcp_connect(uv_connect_t* req,  
                        uv_tcp_t* handle,  
                        const struct sockaddr* addr,  
                        unsigned int addrlen,  
                        uv_connect_cb cb) {  
      int err;  
      int r;  
      
      ...
        
      // 非阻塞发起连接
      connect(uv__stream_fd(handle), addr, addrlen);
  
      // 错误插入pending队列，在事件循环下一次的pending阶段执行
      handle->delayed_error = UV__ERR(ECONNREFUSED);
  
      // 保存回调，例如 此时传入的AfterConnect 回调函数；在libuv连接成功后，会执行该回调函数，即回调C++层
      req->cb = cb;
  
      // 关联起来  
      req->handle = (uv_stream_t*) handle;  
  
      // 注册事件，连接结束后触发，然后执行回调
      uv__io_start(handle->loop, &handle->io_watcher, POLLOUT);
      // ...  
    }  
```

由代码可知，`libuv` 中保存了请求上下文，比如回调，并把 `req` 和 `handle` 做了关联，在执行回调时会使用



**整体结构如下图所示**



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/ConnectWrap.drawio.png)



## 2.3 libuv 回调 C++

当连接结束后，比如完成了三次握手，操作系统会通知 `libuv`，`libuv` 会执行 `uv__stream_connect` 处理连接结果，`uv__stream_connect` 函数如下

```c++
// node-18.15.0/deps/uv/src/unix/stream.c
static void uv__stream_connect(uv_stream_t* stream) {
  int error;
  uv_connect_t* req = stream->connect_req;
  socklen_t errorsize = sizeof(int);
  
  ...
    
  // 获取连接结果
  getsockopt(uv__stream_fd(stream),
             SOL_SOCKET,
             SO_ERROR,
             &error,
             &errorsize);
    error = UV__ERR(error);
  
  ...
    
  // 重点：执行 C++ 层的回调，例如该回调函数是通过 uv__tcp_connect 函数设置进去的，例如 AfterConnect
  if (req->cb)
    req->cb(req, error);
  
  ...
}
```

`uv__stream_connect` 函数从操作系统获取连接结果，然后执行 `C++` 层回调，从前面的分析中可知回调函数就是 `ConnectionWrap` 类的 `AfterConnect` 函数



### 2.3.1 AfterConnect 函数

`AfterConnect` 函数即 `libuv` 回调 `C++` 层的函数，实现如下

```c++
// node-18.15.0/src/connection_wrap.cc
template <typename WrapType, typename UVType>
void ConnectionWrap<WrapType, UVType>::AfterConnect(uv_connect_t* req,
                                                    int status) {
  // 从 libuv 结构体拿到 C++ 层的请求对象  
  BaseObjectPtr<ConnectWrap> req_wrap{static_cast<ConnectWrap*>(req->data)};
  ... 
    
  // 从 C++ 层请求对象拿到对应的 handle 结构体（libuv 里关联起来的），
  // 再通过 handle 拿到对应的C++层 TCPWrap 对象（HandleWrap 关联的） 
  WrapType* wrap = static_cast<WrapType*>(req->handle->data);
  ...
    
  Environment* env = wrap->env();

  HandleScope handle_scope(env->isolate());
  Context::Scope context_scope(env->context());

  ...

  Local<Value> argv[5] = {
    Integer::New(env->isolate(), status),
    wrap->object(),
    req_wrap->object(),
    Boolean::New(env->isolate(), readable),
    Boolean::New(env->isolate(), writable)
  };

  TRACE_EVENT_NESTABLE_ASYNC_END1(TRACING_CATEGORY_NODE2(net, native),
                                  "connect",
                                  req_wrap.get(),
                                  "status",
                                  status);

  // 重点：回调 JS 层 oncomplete ，env 的 oncomplete_string() 定义在PER_ISOLATE_STRING_PROPERTIES宏中，它实际是 oncomplete 属性，所以是调用JS层的oncomplete函数
  req_wrap->MakeCallback(env->oncomplete_string(), arraysize(argv), argv);
}

// node-18.15.0/src/env_properties.h
#define PER_ISOLATE_STRING_PROPERTIES(V)                                       \
  V(oncomplete_string, "oncomplete")                                           \

```

由代码可知，`AfterConnect` 通过它们的关联关系 `req->data` 拿到 `TCPWrap` 对象，最后再通过 `req_wrap` 对象的 `MakeCallback` 函数执行 `JS` 回调函数（`oncomplete` 函数）



## 2.4 C++ 回调 JS

前面也有讲过，`C++` 会通过 `MakeCallback` 函数回调`JS`，此时是回调 `JS` 的 `oncomplete` 函数；`MakeCallback` 函数是由 `AsyncWrap` 类实现，这里再看一下 `MakeCallback` 的定义和实现

```c++
// node-18.15.0/src/async_wrap-inl.h
inline v8::MaybeLocal<v8::Value> AsyncWrap::MakeCallback(
    const v8::Local<v8::String> symbol,
    int argc,
    v8::Local<v8::Value>* argv) {
  return MakeCallback(symbol.As<v8::Name>(), argc, argv);
}

inline v8::MaybeLocal<v8::Value> AsyncWrap::MakeCallback(
    const v8::Local<v8::Symbol> symbol,
    int argc,
    v8::Local<v8::Value>* argv) {
  return MakeCallback(symbol.As<v8::Name>(), argc, argv);
}

// MakeCallback：回调JS
inline v8::MaybeLocal<v8::Value> AsyncWrap::MakeCallback(
    const v8::Local<v8::Name> symbol,
    int argc,
    v8::Local<v8::Value>* argv) {
    
  v8::Local<v8::Value> cb_v;
  
  // 通过 ConnectWrap 的 object() 获取关联的 JS 对象，也就是 JS 创建的 TCPConnectWrap，并获取 oncomplete 属性的值
  if (!object()->Get(env()->context(), symbol).ToLocal(&cb_v))
    return v8::MaybeLocal<v8::Value>();
  
  if (!cb_v->IsFunction()) {
    v8::Isolate* isolate = env()->isolate();
    return Undefined(isolate);
  }
  
  // 接着获取 TCPConnectWrap 对象的 oncomplete 属性的值，值是一个函数，接着调 MakeCallback
  return MakeCallback(cb_v.As<v8::Function>(), argc, argv);
}


// MakeCallback 的实现
// node-18.15.0/src/async_wrap.cc
MaybeLocal<Value> AsyncWrap::MakeCallback(const Local<Function> cb,
                                          int argc,
                                          Local<Value>* argv) {
  ...
  // 调用InternalMakeCallback
  MaybeLocal<Value> ret = InternalMakeCallback(
      env(), object(), object(), cb, argc, argv, context);
  ...
}

// InternalMakeCallback的实现
MaybeLocal<Value> InternalMakeCallback(Environment* env,
                                       Local<Object> resource,
                                       Local<Object> recv,
                                       const Local<Function> callback,
                                       int argc,
                                       Local<Value> argv[],
                                       async_context asyncContext) {
  ...
  // 执行 JS 层回调；最终通过 V8 Function 的 Call 执行该 JS 函数  
  ret = callback->Call(context, recv, argc, argv);
  ...
}
```

`JS` 的 `oncomplete` 函数实际是 `net.js` 中的 `afterConnect()` 函数，其代码如下

```js
// node-18.15.0/src/net.js
const req = new TCPConnectWrap();
req.oncomplete = afterConnect;

function afterConnect(status, handle, req, readable, writable) {
 ...
}
```



# 3 参考

* [Node.js v18.15.0](https://github.com/nodejs/node/tree/v18.15.0)

* [深入剖析 Node.js 底层原理](https://juejin.cn/book/7171733571638738952)

