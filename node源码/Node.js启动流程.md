
# Node的启动流程

> 本文Node的版本为 node-16.18.0

## c++层部分

### 一、Node启动入口

**main入口函数，调用了node::Start开始启动**

* 初始化一个v8实例

* 配置libuv

* 初始化一个Node实例

* 运行Node实例

* 回收v8实例

```cc
// src/node_main.cc
// 入口函数
int main(int argc, char* argv[]) {
  ......
  return node::Start(argc, argv); // 开始启动
}
```

```cc
// src/node.cc
int Start(int argc, char** argv) {
  // 初始化一个v8实例
  InitializationResult result = InitializeOncePerProcess(argc, argv);
  ...
  {
    // libuv事件循环配置
    uv_loop_configure(uv_default_loop(), UV_METRICS_IDLE_TIME);

    // 初始化一个Node.js实例
    NodeMainInstance main_instance(snapshot_data,
                                   uv_default_loop(),
                                   per_process::v8_platform.Platform(),
                                   result.args,
                                   result.exec_args);
    // 运行Node实例
    result.exit_code = main_instance.Run(); 
  }

  // 回收v8实例
  TearDownOncePerProcess();
  return result.exit_code;
}
```

### 二、初始化Node环境和v8实例

1. 先初始化Node环境：通过RegisterBuiltinModules注册内置的c++模块

2. 再初始化v8实例

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106155456.png)

```cc
// src/node.cc
// 初始化一个v8实例
InitializationResult InitializeOncePerProcess(
  int argc,
  char** argv,
  InitializationSettingsFlags flags,
  ProcessFlags::Flags process_flags) {
  ...
  // This needs to run *before* V8::Initialize().
  {
    // 初始化Node环境
    result.exit_code = InitializeNodeWithArgs(
        &(result.args), &(result.exec_args), &errors, process_flags);
  }
  ...
    
  // 初始化v8实例
  per_process::v8_platform.Initialize(
      static_cast<int>(per_process::cli_options->v8_thread_pool_size));
  if (init_flags & kInitializeV8) {
    V8::Initialize();
  }

  performance::performance_v8_start = PERFORMANCE_NOW();
  per_process::v8_initialized = true;

  return result;
}

```

#### 初始化Node环境(注册内c++模块)

1. 初始化node开始运行的时间，调用的是libuv的uv__hrtime

2. 注册内置c++模块

   > 最终调用的是 “_register_模块名字”方法

```c++
// src/node.cc
int InitializeNodeWithArgs(std::vector<std::string>* argv,
                           std::vector<std::string>* exec_argv,
                           std::vector<std::string>* errors,
                           ProcessFlags::Flags flags) {
  ...
  // 初始化node开始运行的时间，实际调用的是libuv的uv__hrtime函数
  per_process::node_start_time = uv_hrtime();

  // 注册内置c++模块  最终调用的是 “_register_模块名字”方法
  // Register built-in modules
  binding::RegisterBuiltinModules();
  ... 
}
```

##### 注册内置c++模块

RegisterBuiltinModules函数的作用：注册内置的c++模块

```cc
// src/node_binding.cc
void RegisterBuiltinModules() {
#define V(modname) _register_##modname();
  NODE_BUILTIN_MODULES(V) // 一个宏
#undef V
}

// NODE_BUILTIN_MODULES宏后展开RegisterBuiltinModules如下
void RegisterBuiltinModules() {  
    #define V(modname) _register_##modname();  
      V(tcp_wrap)   
      V(timers)  
      ...其它模块  
    #undef V  
}  

// 最终展开形态如下
void RegisterBuiltinModules() {  
      _register_tcp_wrap();  // _register_模块名()， 执行了一系列_register开头的函数
      _register_timers();  
}
```

NODE_BUILTIN_MODULES 展开后，为执行了 `_register_模块名()` 的函数。这些函数没有直接存在于Node源码中，而是通过`NODE_MODULE_CONTEXT_AWARE_INTERNAL`宏注册进去的。



c++模块的源码文件最后会调用`NODE_MODULE_CONTEXT_AWARE_INTERNAL`宏，都会被注册到Node的c++模块链表中，这些注册后的模块最终可以通过`process.binding()`获取

> 以tcp_wrap模块为例。 src/tcp_wrap.cc文件里调用了该宏，如 NODE_MODULE_CONTEXT_AWARE_INTERNAL(tcp_wrap, node::TCPWrap::Initialize)



**NODE_MODULE_CONTEXT_AWARE_INTERNAL宏**的定义

```cc
// src/node_binding.h
#define NODE_MODULE_CONTEXT_AWARE_INTERNAL(modname, regfunc)                   \
  NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, nullptr, NM_F_INTERNAL)


#define NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, priv, flags)           \
  static node::node_module _module = {                                         \
      NODE_MODULE_VERSION,                                                     \
      flags,                                                                   \
      nullptr,                                                                 \
      __FILE__,                                                                \
      nullptr,                                                                 \
      (node::addon_context_register_func)(regfunc),                            \
      NODE_STRINGIFY(modname),                                                 \
      priv,                                                                    \
      nullptr};                                                                \
  // "_register_模块名()"函数调了node_module_register，并传入一个node_module数据结构
  void _register_##modname() { node_module_register(&_module); }

```

**node_module_register注册模块函数**

作用：注册内置c++模块到链表中

```cc
// src/node_binding.cc
extern "C" void node_module_register(void* m) {
  struct node_module* mp = reinterpret_cast<struct node_module*>(m);

  if (mp->nm_flags & NM_F_INTERNAL) { // C++内置模块的flag是NM_F_INTERNAL
    mp->nm_link = modlist_internal; // 头插法建立一个单链表
    modlist_internal = mp;
  } else if (!node_is_initialized) {
    // "Linked" modules are included as part of the node project.
    // Like builtins they are registered *before* node::Init runs.
    mp->nm_flags = NM_F_LINKED;
    mp->nm_link = modlist_linked;
    modlist_linked = mp;
  } else {
    thread_local_modpending = mp;
  }
}
```



#### 初始化v8实例

```c++
  /**
   * Initializes V8. This function needs to be called before the first Isolate
   * is created. It always returns true.
   */
  // deps/v8/include/v8.h
  V8_INLINE static bool Initialize() {
    const int kBuildConfiguration =
        (internal::PointerCompressionIsEnabled() ? kPointerCompression : 0) |
        (internal::SmiValuesAre31Bits() ? k31BitSmis : 0) |
        (internal::HeapSandboxIsEnabled() ? kHeapSandbox : 0);
    // 初始化v8实例
    return Initialize(kBuildConfiguration);
  }
```



### 三、初始化并运行Node实例

NodeMainInstance::Run函数里

1. 创建Node执行环境
2. 重载函数Run运行Node

```cc
// src/node_main_instance.cc
int NodeMainInstance::Run() {
  Locker locker(isolate_);
  Isolate::Scope isolate_scope(isolate_);
  HandleScope handle_scope(isolate_);

  int exit_code = 0;
  
  // 创建Node执行环境
  DeleteFnPtr<Environment, FreeEnvironment> env =
      CreateMainEnvironment(&exit_code);
  ...
  Context::Scope context_scope(env->context());
  
  // 继续调用重载的Run函数
  Run(&exit_code, env.get());
  
  return exit_code;
}
```

#### 1、创建Node执行环境

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106155557.png)

```c++
// src/node_main_instance.cc
NodeMainInstance::CreateMainEnvironment(int* exit_code) {
  *exit_code = 0;  // Reset the exit code to 0

  HandleScope handle_scope(isolate_);

  Local<Context> context;
  DeleteFnPtr<Environment, FreeEnvironment> env;

  ...
  context = NewContext(isolate_);
    
  Context::Scope context_scope(context);
  ...
    
  // 重置env变量，初始化上下文
  env.reset(new Environment(isolate_data_.get(),
                              context,
                              args_,
                              exec_args_,
                              nullptr,
                              EnvironmentFlags::kDefaultFlags,
                              {}));

  // 启动初始化
  if (env->RunBootstrapping().IsEmpty()) {
    return nullptr;
  }
  
  return env;
}
```

##### 初始化上下文

```c++
// 初始化执行上下文
// src/env.cc
Environment::Environment(IsolateData* isolate_data,
                         Local<Context> context,
                         const std::vector<std::string>& args,
                         const std::vector<std::string>& exec_args,
                         const EnvSerializeInfo* env_info,
                         EnvironmentFlags::Flags flags,
                         ThreadId thread_id)
    : Environment(isolate_data,
                  context->GetIsolate(),
                  args,
                  exec_args,
                  env_info,
                  flags,
                  thread_id) {
  // 初始化上下文
  InitializeMainContext(context, env_info);
}

// src/env.cc
void Environment::InitializeMainContext(Local<Context> context,
                                        const EnvSerializeInfo* env_info) {
  context_.Reset(context->GetIsolate(), context);
  // 关联context和env 
  AssignToContext(context, ContextInfo(""));
  
  if (env_info != nullptr) {
    DeserializeProperties(env_info);
  } else {
    // 创建其它属性对象 
    CreateProperties();
  }
  ......
}
```

**AssignToContext:关联context和env **

```c++
// src/env.cc
void Environment::AssignToContext(Local<v8::Context> context,
                                  const ContextInfo& info) {
  ContextEmbedderTag::TagNodeContext(context);
  // 保存env对象到context
  context->SetAlignedPointerInEmbedderData(ContextEmbedderIndex::kEnvironment,
                                           this);
  // Used to retrieve bindings
  context->SetAlignedPointerInEmbedderData(
      ContextEmbedderIndex::kBindingListIndex, &(this->bindings_));
  ...
  this->async_hooks()->AddContext(context);
}
```

**CreateProperties:创建其他对象(如process对象)**

```c++
// src/env.cc
void Environment::CreateProperties() {
  HandleScope handle_scope(isolate_);
  Local<Context> ctx = context();

  {
    Context::Scope context_scope(ctx);
    Local<FunctionTemplate> templ = FunctionTemplate::New(isolate());
    templ->InstanceTemplate()->SetInternalFieldCount(
        BaseObject::kInternalFieldCount);
    templ->Inherit(BaseObject::GetConstructorTemplate(this));

    set_binding_data_ctor_template(templ);
  }

  // Store primordials setup by the per-context script in the environment.
  Local<Object> per_context_bindings =
      GetPerContextExports(ctx).ToLocalChecked();
  Local<Value> primordials =
      per_context_bindings->Get(ctx, primordials_string()).ToLocalChecked();
  CHECK(primordials->IsObject());
  set_primordials(primordials.As<Object>());

  Local<String> prototype_string =
      FIXED_ONE_BYTE_STRING(isolate(), "prototype");

#define V(EnvPropertyName, PrimordialsPropertyName)                            \
  {                                                                            \
    Local<Value> ctor =                                                        \
        primordials.As<Object>()                                               \
            ->Get(ctx,                                                         \
                  FIXED_ONE_BYTE_STRING(isolate(), PrimordialsPropertyName))   \
            .ToLocalChecked();                                                 \
    CHECK(ctor->IsObject());                                                   \
    Local<Value> prototype =                                                   \
        ctor.As<Object>()->Get(ctx, prototype_string).ToLocalChecked();        \
    CHECK(prototype->IsObject());                                              \
    set_##EnvPropertyName(prototype.As<Object>());                             \
  }

  V(primordials_safe_map_prototype_object, "SafeMap");
  V(primordials_safe_set_prototype_object, "SafeSet");
  V(primordials_safe_weak_map_prototype_object, "SafeWeakMap");
  V(primordials_safe_weak_set_prototype_object, "SafeWeakSet");
#undef V

  // 创建process对象
  Local<Object> process_object =
      node::CreateProcessObject(this).FromMaybe(Local<Object>());
  // 把process保存到env中
  set_process_object(process_object);
}


// 创建process对象
// src/node_process_object.cc
MaybeLocal<Object> CreateProcessObject(Environment* env) {
  Isolate* isolate = env->isolate();
  EscapableHandleScope scope(isolate);
  Local<Context> context = env->context();

  Local<FunctionTemplate> process_template = FunctionTemplate::New(isolate);
  process_template->SetClassName(env->process_string());
  Local<Function> process_ctor;
  Local<Object> process;
  if (!process_template->GetFunction(context).ToLocal(&process_ctor) ||
      !process_ctor->NewInstance(context).ToLocal(&process)) {
    return MaybeLocal<Object>();
  }

  // process[exiting_aliased_Uint32Array]
  if (process
          ->SetPrivate(context,
                       env->exiting_aliased_Uint32Array(),
                       env->exiting().GetJSArray())
          .IsNothing()) {
    return {};
  }
  
  // 比如 process.version 即查看版本信息
  // process.version
  READONLY_PROPERTY(process,
                    "version",
                    FIXED_ONE_BYTE_STRING(env->isolate(), NODE_VERSION));

  // process.versions
  Local<Object> versions = Object::New(env->isolate());
  READONLY_PROPERTY(process, "versions", versions);

#define V(key)                                                                 \
  if (!per_process::metadata.versions.key.empty()) {                           \
    READONLY_STRING_PROPERTY(                                                  \
        versions, #key, per_process::metadata.versions.key);                   \
  }
  NODE_VERSIONS_KEYS(V)
#undef V

  // process.arch
  READONLY_STRING_PROPERTY(process, "arch", per_process::metadata.arch);

  // process.platform
  READONLY_STRING_PROPERTY(process, "platform", per_process::metadata.platform);

  // process.release
  Local<Object> release = Object::New(env->isolate());
  READONLY_PROPERTY(process, "release", release);
  READONLY_STRING_PROPERTY(release, "name", per_process::metadata.release.name);
  ...
    
  // process._rawDebug: may be overwritten later in JS land, but should be
  // available from the beginning for debugging purposes
  env->SetMethod(process, "_rawDebug", RawDebug);

  return scope.Escape(process);
}
```

##### 启动初始化

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106155834.png)

```cc
// src/node.cc
// 启动初始化
MaybeLocal<Value> Environment::RunBootstrapping() {
  EscapableHandleScope scope(isolate_);

  // 编译加载内置js模块
  // 会执行lib/internal/bootstrap/loaders.js文件
  if (BootstrapInternalLoaders().IsEmpty()) {
    return MaybeLocal<Value>();
  }

  Local<Value> result;
  
  // 加载node.js
  // 会执行lib/internal/bootstrap/node.js文件
  if (!BootstrapNode().ToLocal(&result)) {
    return MaybeLocal<Value>();
  }

  DoneBootstrapping();

  return scope.Escape(result);
}

```

###### 编译并加载内置js模块

```c++
// src/node.cc

// 编译并加载内置js模块，会执行lib/internal/bootstrap/loaders.js文件
MaybeLocal<Value> Environment::BootstrapInternalLoaders() {
  EscapableHandleScope scope(isolate_);

  // 函数的形参列表：字符串数组
  // Create binding loaders
  std::vector<Local<String>> loaders_params = {
      process_string(),
      FIXED_ONE_BYTE_STRING(isolate_, "getLinkedBinding"),
      FIXED_ONE_BYTE_STRING(isolate_, "getInternalBinding"),
      primordials_string()}; // 加载primordials.js
  
  // 对象数组
  std::vector<Local<Value>> loaders_args = {
      process_object(),
      // getLinkedBinding根据模块名从链表中获取到对应的模块
      // Node在C++层对外暴露了AddLinkedBinding函数注册模块，Node维护了一个单独的链表保存这些模块，最终在js层可通过process._linkedBinding函数获取
      NewFunctionTemplate(binding::GetLinkedBinding) 
          ->GetFunction(context())
          .ToLocalChecked(),
    // getInternalBinding也是根据模块名从链表中获取到对应的模块
    // Node也一样维护了一个链表存C++内置模块，最终在js层可通过process.binding函数获取
      NewFunctionTemplate(binding::GetInternalBinding) 
          ->GetFunction(context())
          .ToLocalChecked(),
      primordials()}; 

  // Bootstrap internal loaders
  Local<Value> loader_exports;
  
  // 编译并执行internal/bootstrap/loaders.js文件，然后调用编译好的方法
  if (!ExecuteBootstrapper(
           this, "internal/bootstrap/loaders", &loaders_params, &loaders_args)
           .ToLocal(&loader_exports)) {
    return MaybeLocal<Value>();
  }
  
  // 获取到从loaders.js文件导出到internalBinding函数，该函数用于加载内置C++模块函数
  // internal_binding_string()即internalBinding，见src/env.h文件里的V(internal_binding_string, "internalBinding")
  Local<Object> loader_exports_obj = loader_exports.As<Object>();
  Local<Value> internal_binding_loader =
      loader_exports_obj->Get(context(), internal_binding_string())
          .ToLocalChecked();
  set_internal_binding_loader(internal_binding_loader.As<Function>());
  
  // 获取到从loaders.js文件导出的require函数，即loaders.js的nativeModuleRequire函数，用于加载内置js模块
  // require_string()即require，见src/env.h文件里的V(require_string, "require")               
  Local<Value> require =
      loader_exports_obj->Get(context(), require_string()).ToLocalChecked(); 
  set_native_module_require(require.As<Function>());

  return scope.Escape(loader_exports);
}
```

加载内置c++模块的实现

```c++
// src/node_binding.cc
// 根据模块名从这个链表中找到对应的模块
void GetInternalBinding(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  CHECK(args[0]->IsString());

  Local<String> module = args[0].As<String>();
  node::Utf8Value module_v(env->isolate(), module);
  Local<Object> exports;

  node_module* mod = FindModule(modlist_internal, *module_v, NM_F_INTERNAL);
  if (mod != nullptr) {
    // 找不到就初始化模块
    exports = InitModule(env, mod, module);
    env->internal_bindings.insert(mod);
  } else if (!strcmp(*module_v, "constants")) {
    exports = Object::New(env->isolate());
    CHECK(
        exports->SetPrototype(env->context(), Null(env->isolate())).FromJust());
    DefineConstants(env->isolate(), exports);
  } else if (!strcmp(*module_v, "natives")) {
    // GetSourceObject获取模块的源码
    exports = native_module::NativeModuleEnv::GetSourceObject(env->context());
    // Legacy feature: process.binding('natives').config contains stringified
    // config.gypi
    CHECK(exports
              ->Set(env->context(),
                    env->config_string(),
                    native_module::NativeModuleEnv::GetConfigString(
                        env->isolate()))
              .FromJust());
  } else {
    return THROW_ERR_INVALID_MODULE(env, "No such module: %s", *module_v);
  }

  args.GetReturnValue().Set(exports);
}
```

加载内置js模块的实现

```c++
// src/node_binding.cc
void GetInternalBinding(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  Local<String> module = args[0].As<String>();
  node::Utf8Value module_v(env->isolate(), module);
  Local<Object> exports;

  // 查找内置js模块
  node_module* mod = FindModule(modlist_internal, *module_v, NM_F_INTERNAL);
  if (mod != nullptr) {
    exports = InitModule(env, mod, module);
    env->internal_bindings.insert(mod); // 插入该模块
  } else if (!strcmp(*module_v, "constants")) {
    exports = Object::New(env->isolate());
    CHECK(
        exports->SetPrototype(env->context(), Null(env->isolate())).FromJust());
    DefineConstants(env->isolate(), exports);
  } else if (!strcmp(*module_v, "natives")) {
    // 获取模块的js源码
    exports = native_module::NativeModuleEnv::GetSourceObject(env->context());
    // Legacy feature: process.binding('natives').config contains stringified
    // config.gypi
    CHECK(exports
              ->Set(env->context(),
                    env->config_string(),
                    native_module::NativeModuleEnv::GetConfigString(
                        env->isolate()))
              .FromJust());
  } else {
    return THROW_ERR_INVALID_MODULE(env, "No such module: %s", *module_v);
  }

  // 返回exports对象
  args.GetReturnValue().Set(exports);
}

// 获取内置js模块的源码
// src/node_native_module.cc
Local<Object> NativeModuleLoader::GetSourceObject(Local<Context> context) {
  Isolate* isolate = context->GetIsolate();
  Local<Object> out = Object::New(isolate);
  for (auto const& x : source_) { // source_变量是LoadJavaScriptSource函数加载数据进去的
    Local<String> key = OneByteString(isolate, x.first.c_str(), x.first.size());
    out->Set(context, key, x.second.ToStringChecked(isolate)).FromJust();
  }
  return out;
}

// src/node_native_module.h

// Node使用js2c.py文件编译加载数据到node_javascript.cc文件，里面也加载了数据到source_
// Generated by tools/js2c.py as node_javascript.cc
void LoadJavaScriptSource();  // Loads data into source_
```



###### 初始化Node

```c++
// src/node.cc
MaybeLocal<Value> Environment::BootstrapNode() {
  EscapableHandleScope scope(isolate_);

  // process, require, internalBinding, primordials
  std::vector<Local<String>> node_params = {
      process_string(),
      require_string(),
      internal_binding_string(),
      primordials_string()};
  std::vector<Local<Value>> node_args = {
      process_object(),
      native_module_require(),  // 内置js模块导入
      internal_binding_loader(), // 内置C++模块加载器
      primordials()};

  // 编译并执行internal/bootstrap/node.js文件
  MaybeLocal<Value> result = ExecuteBootstrapper(
      this, "internal/bootstrap/node", &node_params, &node_args);

  // 线程初始化
  // TODO(joyeecheung): skip these in the snapshot building for workers.
  auto thread_switch_id =
      is_main_thread() ? "internal/bootstrap/switches/is_main_thread"
                       : "internal/bootstrap/switches/is_not_main_thread";
  result =
      ExecuteBootstrapper(this, thread_switch_id, &node_params, &node_args);
  
  ...
}

```



#### 2、重载函数Run运行Node

```cc
// src/node_main_instance.cc
// Run的重载函数
void NodeMainInstance::Run(int* exit_code, Environment* env) {
  if (*exit_code == 0) {
    // 加载Node环境, 在里面初始化libuv：InitializeLibuv
    LoadEnvironment(env, StartExecutionCallback{});

    // 进入libuv事件循环
    *exit_code = SpinEventLoop(env).FromMaybe(1);
  }
  ......
}
```

##### 加载Node环境

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106155933.png)

```c++
// src/api/environment.cc
MaybeLocal<Value> LoadEnvironment(
    Environment* env,
    StartExecutionCallback cb) {
  // 初始化libuv
  env->InitializeLibuv(); 
  env->InitializeDiagnostics();

  // 加载Node执行环境
  // 最终会通过ExecuteBootstrapper执行internal/main目录下各种js文件
  // 如加载文件internal/main/run_main_module.js
  return StartExecution(env, cb);
}
```

###### 初始化libuv

```c++
// src/env.cc
void Environment::InitializeLibuv() {
  HandleScope handle_scope(isolate());
  Context::Scope context_scope(context());

  // timer_handle，实现Node.js中定时器的数据结构，对应Libuv的time阶段
  CHECK_EQ(0, uv_timer_init(event_loop(), timer_handle()));
  uv_unref(reinterpret_cast<uv_handle_t*>(timer_handle()));

  // immediate_check_handle，实现Node.js中setImmediate的数据结构，对应Libuv的check阶段
  CHECK_EQ(0, uv_check_init(event_loop(), immediate_check_handle())); 
  uv_unref(reinterpret_cast<uv_handle_t*>(immediate_check_handle()));

  CHECK_EQ(0, uv_idle_init(event_loop(), immediate_idle_handle()));

  CHECK_EQ(0, uv_check_start(immediate_check_handle(), CheckImmediate));

  // Inform V8's CPU profiler when we're idle.  The profiler is sampling-based
  // but not all samples are created equal; mark the wall clock time spent in
  // epoll_wait() and friends so profiling tools can filter it out.  The samples
  // still end up in v8.log but with state=IDLE rather than state=EXTERNAL.
  CHECK_EQ(0, uv_prepare_init(event_loop(), &idle_prepare_handle_));
  CHECK_EQ(0, uv_check_init(event_loop(), &idle_check_handle_));

  // task_queues_async_ 用于子线程和主线程通信
  CHECK_EQ(0, uv_async_init(
      event_loop(),
      &task_queues_async_,
      [](uv_async_t* async) {
        Environment* env = ContainerOf(
            &Environment::task_queues_async_, async);
        HandleScope handle_scope(env->isolate());
        Context::Scope context_scope(env->context());
        env->RunAndClearNativeImmediates();
      }));
  uv_unref(reinterpret_cast<uv_handle_t*>(&idle_prepare_handle_));
  uv_unref(reinterpret_cast<uv_handle_t*>(&idle_check_handle_));
  uv_unref(reinterpret_cast<uv_handle_t*>(&task_queues_async_));

  {
    Mutex::ScopedLock lock(native_immediates_threadsafe_mutex_);
    task_queues_async_initialized_ = true;
    if (native_immediates_threadsafe_.size() > 0 ||
        native_immediates_interrupts_.size() > 0) {
      uv_async_send(&task_queues_async_);
    }
  }

  ...
}
```



###### 加载Node执行环境

```c++
// src/node.cc
MaybeLocal<Value> StartExecution(Environment* env, StartExecutionCallback cb) {
  InternalCallbackScope callback_scope(
      env,
      Object::New(env->isolate()),
      { 1, 0 },
      InternalCallbackScope::kSkipAsyncHooks);

  if (cb != nullptr) {
    EscapableHandleScope scope(env->isolate());

    if (StartExecution(env, "internal/bootstrap/environment").IsEmpty())
      return {};
    ...
    return scope.EscapeMaybe(cb(info));
  }

  // TODO(joyeecheung): move these conditions into JS land and let the
  // deserialize main function take precedence. For workers, we need to
  // move the pre-execution part into a different file that can be
  // reused when dealing with user-defined main functions.
  if (!env->snapshot_deserialize_main().IsEmpty()) {
    return env->RunSnapshotDeserializeMain();
  }

  if (env->worker_context() != nullptr) {
    return StartExecution(env, "internal/main/worker_thread");
  }

  if (first_argv == "inspect") {
    return StartExecution(env, "internal/main/inspect");
  }
 ...

  if (!first_argv.empty() && first_argv != "-") {
    return StartExecution(env, "internal/main/run_main_module");
  }
  ...
  return StartExecution(env, "internal/main/eval_stdin");
}
```

##### 开启libuv事件循环

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106155957.png)



用户的JS代码会往Libuv里注册一些任务，在Libuv的事件循环中，会进行一轮又一轮的事件循环处理，直到没有需要处理的任务，Libuv会退出，从而Node.js退出。

```c++
// libuv事件循环
// src/api/embed_heplers.ts
Maybe<int> SpinEventLoop(Environment* env) {
  CHECK_NOT_NULL(env);
  MultiIsolatePlatform* platform = GetMultiIsolatePlatform(env);
  CHECK_NOT_NULL(platform);

  Isolate* isolate = env->isolate();
  HandleScope handle_scope(isolate);
  Context::Scope context_scope(env->context());
  SealHandleScope seal(isolate);

  if (env->is_stopping()) return Nothing<int>();

  env->set_trace_sync_io(env->options()->trace_sync_io);
  {
    bool more;
    env->performance_state()->Mark(
        node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_START);
    do {
      if (env->is_stopping()) break;
      
      // 启动libuv事件循环，调用libuv的事件循环，处理事件循环的7个阶段
      uv_run(env->event_loop(), UV_RUN_DEFAULT);
      if (env->is_stopping()) break;

      platform->DrainTasks(isolate);

      // 事件循环是否还有待处理事件
      more = uv_loop_alive(env->event_loop());
      
      if (more && !env->is_stopping()) continue;

      if (EmitProcessBeforeExit(env).IsNothing())
        break;

      {
        HandleScope handle_scope(isolate);
        if (env->RunSnapshotSerializeCallback().IsEmpty()) {
          break;
        }
      }

      // Emit `beforeExit` if the loop became alive either after emitting
      // event, or after running some callbacks.
      more = uv_loop_alive(env->event_loop());
    } while (more == true && !env->is_stopping());
    env->performance_state()->Mark(
        node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_EXIT);
  }
  ...
    
  // 没有待处理事件，则退出 process.emit('exit')  
  return EmitProcessExit(env);
}
```



## js层部分

### 加载loaders.js文件

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106160207.png)



该文件主要包括

1. c++模块的加载逻辑

2. 内置js模块的加载逻辑(NativeModule)

3. 导出loaderExports对象给c++层使用

   

**process.binding函数加载内置c++模块**

```js
// lib/internal/bootstrap/loaders.js

// 1、 internalBinding() 加载内置c++模块
let internalBinding;
{
  const bindingObj = ObjectCreate(null);
  // eslint-disable-next-line no-global-assign
  internalBinding = function internalBinding(module) {
    let mod = bindingObj[module];
    if (typeof mod !== 'object') {
      // 调用c++层的getInternalBinding封装，node_binding.cc的GetInternalBinding
      mod = bindingObj[module] = getInternalBinding(module);
      ArrayPrototypePush(moduleLoadList, `Internal Binding ${module}`);
    }
    return mod;
  };
}

// 2、process.binding() 加载内置c++模块
// 3、process._linkedBinding() 加载嵌入式的c++模块
{
  const bindingObj = ObjectCreate(null);

  // 在process对象中挂载binding函数，主要用于内置的JS模块，该函数主要逻辑是根据模块名查找对应的C++模块
  process.binding = function binding(module) {
    module = String(module);
    // internalBindingAllowlist配置了内置的c++模块名
    if (internalBindingAllowlist.has(module)) {
      if (runtimeDeprecatedList.has(module)) {
        runtimeDeprecatedList.delete(module);
        // 输出警告日志
        process.emitWarning(
          `Access to process.binding('${module}') is deprecated.`,
          'DeprecationWarning',
          'DEP0111');
      }
      
      // 遗留的模块使用不同的加载方式，如util模块
      if (legacyWrapperList.has(module)) {
        return nativeModuleRequire('internal/legacy/processbinding')[module]();
      }
      
      // 加载内置的c++模块
      return internalBinding(module);
    }
    // eslint-disable-next-line no-restricted-syntax
    throw new Error(`No such module: ${module}`);
  };

  process._linkedBinding = function _linkedBinding(module) {
    module = String(module);
    let mod = bindingObj[module];
    if (typeof mod !== 'object')
      // 调用的是node_binding.cc的GetLinkedBinding
      mod = bindingObj[module] = getLinkedBinding(module); 
    return mod;
  };
}
```

**NativeModule**

>  源码中lib文件夹下的js文件即Node自带的原生js模块

1. NativeModule负责内置JS模块的加载，即编译和执行

   > node.gyp配置文件有一项配置是node_js2c，作用是用python去调用一个名为tools/js2c.py的文件，内置JS模块的代码最终转成字符存在node_javascript.cc文件的，由js2c.py文件调用LoadJavaScriptSource函数加载内置js模块源码

2. 提供一个require函数，加载内置JS模块

```js
// lib/internal/bootstrap/loaders.js
class NativeModule {
  /**
   * A map from the module IDs to the module instances.
   * @type {Map<string, NativeModule>}
   */
  // moduleIds保存了原生JS模块的名称列表，即js文件名
  static map = new SafeMap(
    ArrayPrototypeMap(moduleIds, (id) => [id, new NativeModule(id)])
  );
......
}
```

**导出loaderExports给c++层使用**

```js
// lib/internal/bootstrap/loaders.js

// 执行完loader.js后，返回下面这3个变量给c++层使用
// Think of this as module.exports in this file even though it is not
// written in CommonJS style.
const loaderExports = {
  internalBinding, // 获取内建c++模块
  NativeModule, // 内建js模块的类的信息
  require: nativeModuleRequire // 导入内置js模块
};

// require函数的实现
function nativeModuleRequire(id) {
  if (id === loaderId) {
    return loaderExports;
  }
  const mod = NativeModule.map.get(id);
  // Can't load the internal errors module from here, have to use a raw error.
  // eslint-disable-next-line no-restricted-syntax
  if (!mod) throw new TypeError(`Missing internal module '${id}'`);
  
  return mod.compileForInternalLoader();
}

// lib/internal/bootstrap/loaders.js
compileForInternalLoader() {
    if (this.loaded || this.loading) {
      return this.exports;
    }

    const id = this.id;
    this.loading = true;

    try {
      const requireFn = StringPrototypeStartsWith(this.id, 'internal/deps/') ?
        requireWithFallbackInDeps : nativeModuleRequire;

      // 编译, compileFunction是native_module模块的方法
      const fn = compileFunction(id);
      // 编译加载模块
      fn(this.exports, requireFn, this, process, internalBinding, primordials);
      // 此时模块已加载完成
      this.loaded = true;
    } finally {
      this.loading = false;
    }

    ArrayPrototypePush(moduleLoadList, `NativeModule ${id}`);
    return this.exports;
  }

// src/node_native_module_env.cc
// CompileFunction方法
void NativeModuleEnv::CompileFunction(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  CHECK(args[0]->IsString());
  node::Utf8Value id_v(env->isolate(), args[0].As<String>());
  const char* id = *id_v;
  NativeModuleLoader::Result result;
  
  // 编译模块
  MaybeLocal<Function> maybe =
      NativeModuleLoader::GetInstance()->CompileAsModule(
          env->context(), id, &result);
  RecordResult(id, result, env);
  Local<Function> fn;
  if (maybe.ToLocal(&fn)) {
    args.GetReturnValue().Set(fn);
  }
}
// src/node_native_module_env.cc
// 编译模块
MaybeLocal<Function> NativeModuleLoader::CompileAsModule(
    Local<Context> context,
    const char* id,
    NativeModuleLoader::Result* result) {
  Isolate* isolate = context->GetIsolate();
  std::vector<Local<String>> parameters = {
      FIXED_ONE_BYTE_STRING(isolate, "exports"),
      FIXED_ONE_BYTE_STRING(isolate, "require"),
      FIXED_ONE_BYTE_STRING(isolate, "module"),
      FIXED_ONE_BYTE_STRING(isolate, "process"),
      FIXED_ONE_BYTE_STRING(isolate, "internalBinding"),
      FIXED_ONE_BYTE_STRING(isolate, "primordials")};
  // 查找和编译
  return LookupAndCompile(context, id, &parameters, result);
}

// src/node_native_module_env.cc

// Returns Local<Function> of the compiled module if return_code_cache
// is false (we are only compiling the function).
// Otherwise return a Local<Object> containing the cache.
MaybeLocal<Function> NativeModuleLoader::LookupAndCompile(
    Local<Context> context,
    const char* id,
    std::vector<Local<String>>* parameters,
    NativeModuleLoader::Result* result) {
  Isolate* isolate = context->GetIsolate();
  EscapableHandleScope scope(isolate);

  Local<String> source;
  if (!LoadBuiltinModuleSource(isolate, id).ToLocal(&source)) {
    return {};
  }

  std::string filename_s = std::string("node:") + id;
  Local<String> filename =
      OneByteString(isolate, filename_s.c_str(), filename_s.size());
  ScriptOrigin origin(isolate, filename, 0, 0, true);

  ScriptCompiler::CachedData* cached_data = nullptr;
  {
    // Note: The lock here should not extend into the
    // `CompileFunctionInContext()` call below, because this function may
    // recurse if there is a syntax error during bootstrap (because the fatal
    // exception handler is invoked, which may load built-in modules).
    Mutex::ScopedLock lock(code_cache_mutex_);
    auto cache_it = code_cache_.find(id);
    if (cache_it != code_cache_.end()) {
      // Transfer ownership to ScriptCompiler::Source later.
      cached_data = cache_it->second.release();
      code_cache_.erase(cache_it);
    }
  }

  const bool has_cache = cached_data != nullptr;
  ScriptCompiler::CompileOptions options =
      has_cache ? ScriptCompiler::kConsumeCodeCache
                : ScriptCompiler::kEagerCompile;
  ScriptCompiler::Source script_source(source, origin, cached_data);

  // 编译
  MaybeLocal<Function> maybe_fun =
      ScriptCompiler::CompileFunctionInContext(context,
                                               &script_source,
                                               parameters->size(),
                                               parameters->data(),
                                               0,
                                               nullptr,
                                               options);

  // This could fail when there are early errors in the native modules,
  // e.g. the syntax errors
  Local<Function> fun;
  if (!maybe_fun.ToLocal(&fun)) {
    // In the case of early errors, v8 is already capable of
    // decorating the stack for us - note that we use CompileFunctionInContext
    // so there is no need to worry about wrappers.
    return MaybeLocal<Function>();
  }

  // XXX(joyeecheung): this bookkeeping is not exactly accurate because
  // it only starts after the Environment is created, so the per_context.js
  // will never be in any of these two sets, but the two sets are only for
  // testing anyway.

  *result = (has_cache && !script_source.GetCachedData()->rejected)
                ? Result::kWithCache
                : Result::kWithoutCache;
  // Generate new cache for next compilation
  std::unique_ptr<ScriptCompiler::CachedData> new_cached_data(
      ScriptCompiler::CreateCodeCacheForFunction(fun));
  CHECK_NOT_NULL(new_cached_data);

  {
    Mutex::ScopedLock lock(code_cache_mutex_);
    // The old entry should've been erased by now so we can just emplace.
    // If another thread did the same thing in the meantime, that should not
    // be an issue.
    code_cache_.emplace(id, std::move(new_cached_data));
  }

  return scope.Escape(fun);
}
```



### 加载node.js文件

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106160243.png)

```js
// lib/internal/bootstrap/node.js

// 设置process对象
setupProcessObject();
// 设置global对象
setupGlobalProxy();
// 设置buffer对象
setupBuffer();
...

// 初始化main线程和worker线程
// Bootstrappers for all threads, including worker threads and main thread
const perThreadSetup = require('internal/process/per_thread');

// 设置一些process变量
const rawMethods = internalBinding('process_methods');
process.dlopen = rawMethods.dlopen; // 加载.node模块
process.uptime = rawMethods.uptime;
...

// 使用v8的任务队列，如process.nextTick
const {
  setupTaskQueue,
  queueMicrotask
} = require('internal/process/task_queues');
...

// 设置全局变量
// https://html.spec.whatwg.org/multipage/webappapis.html#windoworworkerglobalscope
const timers = require('timers');
defineOperation(globalThis, 'clearInterval', timers.clearInterval);
defineOperation(globalThis, 'clearTimeout', timers.clearTimeout);
defineOperation(globalThis, 'setInterval', timers.setInterval);
defineOperation(globalThis, 'setTimeout', timers.setTimeout);
defineOperation(globalThis, 'queueMicrotask', queueMicrotask);
...
// Non-standard extensions:
defineOperation(globalThis, 'clearImmediate', timers.clearImmediate);
defineOperation(globalThis, 'setImmediate', timers.setImmediate);
...

const {
    processImmediate, // 处理immediate任务
    processTimers, // 处理定时器任务
  } = internalTimers.getTimerCallbacks(runNextTicks);

// 预加载一些模块
require('fs');
require('v8');
require('vm');
require('url');
require('internal/options');
```



### 加载 primordials.js文件

```js
// 创建一个全局代理，用于访问js内建的方法
// lib/internal/per_context/primordials.js

function copyPropsRenamed(src, dest, prefix) {
  for (const key of ReflectOwnKeys(src)) {
    const newKey = getNewKey(key);
    const desc = ReflectGetOwnPropertyDescriptor(src, key);
    if ('get' in desc) {
      copyAccessor(dest, prefix, newKey, desc);
    } else {
      const name = `${prefix}${newKey}`;
      ReflectDefineProperty(dest, name, { __proto__: null, ...desc });
      if (varargsMethods.includes(name)) {
        ReflectDefineProperty(dest, `${name}Apply`, {
          __proto__: null,
          // `src` is bound as the `this` so that the static `this` points
          // to the object it was defined on,
          // e.g.: `ArrayOfApply` gets a `this` of `Array`:
          value: applyBind(desc.value, src),
        });
      }
    }
  }
}

// 冻结primordials属性，防止修改primordials
ObjectFreeze(primordials);
```



### 加载run_main_module.js文件

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221106160323.png)



1. 做一些主线程启动前的初始化
2. cjs模块加载，即调用executeUserEntryPoint函数

```js
// src/internal/main/run_main_module.js
'use strict';

const { RegExpPrototypeExec } = primordials;

const {
  prepareMainThreadExecution
} = require('internal/bootstrap/pre_execution');

prepareMainThreadExecution(true); // 做一些主线程启动前的准备

markBootstrapComplete();

// Necessary to reset RegExp statics before user code runs.
RegExpPrototypeExec(/^/, '');

// Note: this loads the module through the ESM loader if the module is
// determined to be an ES module. This hangs from the CJS module loader
// because we currently allow monkey-patching of the module loaders
// in the preloaded scripts through require('module').
// runMain here might be monkey-patched by users in --require.
// XXX: the monkey-patchability here should probably be deprecated.

//  cjs模块加载，即调用executeUserEntryPoint函数 lib/internal/modules/run_main.js
require('internal/modules/cjs/loader').Module.runMain(process.argv[1]);
```



**主线程启动前的初始化**

```js
// lib/internal/bootstrap/pre_execution.js\
function prepareMainThreadExecution(expandArgv1 = false,
                                    initialzeModules = true) {
  refreshRuntimeOptions();
  ...
  // Patch the process object with legacy properties and normalizations
  setupTraceCategoryState();
  setupPerfHooks();
  setupInspectorHooks();
  setupWarningHandler();
  setupFetch();
  setupWebCrypto();
  setupCustomEvent();

  // Resolve the coverage directory to an absolute path, and
  // overwrite process.env so that the original path gets passed
  // to child processes even when they switch cwd.
  if (process.env.NODE_V8_COVERAGE) {
    process.env.NODE_V8_COVERAGE =
      setupCoverageHooks(process.env.NODE_V8_COVERAGE);
  }

  setupDebugEnv();

  // Print stack trace on `SIGINT` if option `--trace-sigint` presents.
  setupStacktracePrinterOnSigint();

  // Process initial diagnostic reporting configuration, if present.
  initializeReport();
  initializeReportSignalHandlers();  // Main-thread-only.

  initializeHeapSnapshotSignalHandlers();

  // If the process is spawned with env NODE_CHANNEL_FD, it's probably
  // spawned by our child_process module, then initialize IPC.
  // This attaches some internal event listeners and creates:
  // process.send(), process.channel, process.connected,
  // process.disconnect().
  setupChildProcessIpcChannel();

  // Load policy from disk and parse it.
  initializePolicy();

  // If this is a worker in cluster mode, start up the communication
  // channel. This needs to be done before any user code gets executed
  // (including preload modules).
  initializeClusterIPC();

  initializeSourceMapsHandlers();
  initializeDeprecations();
  initializeWASI();

  require('internal/v8/startup_snapshot').runDeserializeCallbacks();

  if (!initialzeModules) {
    return;
  }

  initializeCJSLoader(); // 初始化cjs模块加载器
  initializeESMLoader(); // 初始化esm模块加载器
 
}

// 初始化cjs加载器
function initializeCJSLoader() {
  const CJSLoader = require('internal/modules/cjs/loader');
  if (!getEmbedderOptions().noGlobalSearchPaths) {
    CJSLoader.Module._initPaths();
  }

  // TODO(joyeecheung): deprecate this in favor of a proper hook?
  
  // runMain加载用户对js代码，即load方法加载用户的模块
  CJSLoader.Module.runMain =
    require('internal/modules/run_main').executeUserEntryPoint;
}


// For backwards compatibility, we have to run a bunch of
// monkey-patchable code that belongs to the CJS loader (exposed by
// `require('module')`) even when the entry point is ESM.
function executeUserEntryPoint(main = process.argv[1]) {
  const resolvedMain = resolveMainPath(main);
  const useESMLoader = shouldUseESMLoader(resolvedMain);
  if (useESMLoader) {
    runMainESM(resolvedMain || main);
  } else {
    // Module._load is the monkey-patchable CJS module loader.
    // true表示是入口模块
    Module._load(main, null, true);
  }
}

// 处理进程间通信
function setupChildProcessIpcChannel() {
  if (process.env.NODE_CHANNEL_FD) {
    const assert = require('internal/assert');

    // 环境变量NODE_CHANNEL_FD是在创建子进程的时候设置的，如果有说明当前启动的进程是子进程，则需要处理进程间通信
    const fd = NumberParseInt(process.env.NODE_CHANNEL_FD, 10);
    assert(fd >= 0);

    // Make sure it's not accidentally inherited by child processes.
    delete process.env.NODE_CHANNEL_FD;

    
    const serializationMode =
      process.env.NODE_CHANNEL_SERIALIZATION_MODE || 'json';
    delete process.env.NODE_CHANNEL_SERIALIZATION_MODE;

    require('child_process')._forkChild(fd, serializationMode);
    assert(process.send);
  }
}

// 处理cluster模块的进程间通信
function initializeClusterIPC() {
  if (process.argv[1] && process.env.NODE_UNIQUE_ID) {
    const cluster = require('cluster');
    cluster._setupWorker();
    // Make sure it's not accidentally inherited by child processes.
    delete process.env.NODE_UNIQUE_ID;
  }
}
```



