# 1 前言

`Egg.js` 内部实现了一套自动加载机制，它能够自动加载约定好的目录文件。我们自己也可以基于 `Egg` 封装一个有自己加载风格的框架，本文主要结合源码讲下 `Egg` 的加载器，以及 `Egg` 的生命周期，如有错误之处，欢迎指正。



**此文所涉及源码库的版本**

* [egg](https://link.zhihu.com/?target=https%3A//github.com/eggjs/egg)： 3.23.0
* [egg-core](https://link.zhihu.com/?target=https%3A//github.com/eggjs/egg-core): 5.4.1
* [egg-cluster](https://link.zhihu.com/?target=https%3A//github.com/eggjs/egg-cluster) : 2.2.1
* [ready-callback](https://github.com/node-modules/ready-callback)：3.0.0
* [get-ready](https://www.npmjs.com/package/get-ready)：2.0.1



# 2 Application 和 Agent 类

`Egg` 包对外提供了 `Application` 类和 `Agent` 类，这两个类默认在 `App Worker` 进程和 `Agent` 进程初始化时会实例化的。

> 例如 `egg-scripts` 启动程序在调用 `egg-cluster` 时，没传 `framework` 参数时，默认是读取的 `egg` 包的 `Application` 类和 `Agent` 类（关于多进程的知识可以参考这篇[文章](https://zhuanlan.zhihu.com/p/699815241)）

```js
// app worker 实例化 Application 类
// egg-cluster/lib/app_worker.js
const Application = require(options.framework).Application;
app = new Application(options);

// agent worker 实例化 Agent 类
// egg-cluster/lib/agent_worker.js
const Agent = require(options.framework).Agent;
agent = new Agent(options);
```

下面是 `egg` 包对外暴露的 `Application` 类和 `Agent` 类和各自的加载器类

```js
// egg/index.js
'use strict';

// ......
/**
 * @member {Application} Egg#Application
 * @since 1.0.0
 */
exports.Application = require('./lib/application');

/**
 * @member {Agent} Egg#Agent
 * @since 1.0.0
 */
exports.Agent = require('./lib/agent');

/**
 * @member {AppWorkerLoader} Egg#AppWorkerLoader
 * @since 1.0.0
 */
exports.AppWorkerLoader = require('./lib/loader').AppWorkerLoader;

/**
 * @member {AgentWorkerLoader} Egg#AgentWorkerLoader
 * @since 1.0.0
 */
exports.AgentWorkerLoader = require('./lib/loader').AgentWorkerLoader;

// ......
```



**Agent 和 Application 类的继承关系如下**

* `Agent` 和 `Application` 类都继承自 `EggApplication` 类

* `EggApplication` 类继承自 `EggCore` 类
* `EggCore` 类继承自 `KoaApplication(即koa)`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/egg%E7%B1%BB%E5%9B%BE11.png)



## 2.1 EggCore 基类

`EggCore` 是 `Egg` 中的基类，所有 `Egg` 应用类最终都继承它，它继承了 `Koa` ，它的逻辑如下

1、挂载一系列对象

* 比如项目中我们继承的 `Controller` 和 `Service` 类其实是 `BaseContextClass` 类，可以在 `Controller`和 `Service` 中使用 `this.ctx`、`this.app`、`this.config`、`this.service` 等对象

2、实例化生命周期对象 `lifecycle`，所以 `Egg` 有一套公共的生命周期实现

3、实例化加载器 `Loader`

* 如果自己基于 `Egg` 又封装了一个上层框架，那也可以实现自己的加载器，并实现加载器的加载方法； `Egg` 默认自带的 `Application` 和 `Agent` 都有各自的加载器实现



**EggCore 源码如下**

> egg-core/lib/egg.js

```js
class EggCore extends KoaApplication {
    constructor(ctx) {
        super();

        // 平时继承的 Controller 和 Service 类其实是 BaseContextClass 类，
        // 所以可以在Controller和Service中使用this.ctx对象
        this.BaseContextClass = BaseContextClass;
        const Controller = this.BaseContextClass;
        this.Controller = Controller;
        const Service = this.BaseContextClass;
        this.Service = Service;

        // 1. 实例化生命周期
        this.lifecycle = new Lifecycle({
            baseDir: options.baseDir,
            app: this,
            logger: this.console,
        });

        // 2. 实例化加载器
        // egg 使用了 Symbol.for('egg#loader') 变量存放具体的加载器实现
        const Loader = this[EGG_LOADER];
        this.loader = new Loader({
            baseDir: options.baseDir,
            app: this,
            plugins: options.plugins,
            logger: this.console,
            serverScope: options.serverScope,
            env: options.env,
        });
    }
}
```

> egg-core/lib/utils/base_context_class.js 的 BaseContextClass 类

```js
class BaseContextClass { 
    // 该实现即平时使用this.ctx、this.app、this.config、this.service，而没有this.controller的原因
    constructor(ctx) {   
        this.ctx = ctx;
        this.app = ctx.app;
        this.config = ctx.app.config;
        this.service = ctx.service; 
    }
}
```



## 2.2 EggApplicaion 应用基类

`EggApplication` 类继承自 `EggCore`，它也是 `Agent` 类和 `Application` 类的父类，它的逻辑如下

1、调用加载器的 `loadConfig()` 加载配置

2、实例化 `App` 和 `Agent` 之间的 `messenger` 通信对象，这个对象提供了 `Agent` 和 `App` 进程之间方便通信的一些 `API`

3、监听 `egg-ready` 事件，里面会触发生命周期"应用启动完成"函数（ `triggerServerDidReady()`）

* 当 `egg` 所有 `App` 进程启动完毕后，会发 `egg-ready` 事件通知各个进程，此时每个 `App` 进程监听到 `egg-ready` 事件后就会开始触发生命周期函数的“应用启动完成”阶段

4、注册一些方法



**EggApplication 源码如下**

> egg/lib/egg.js

```js
// agent和application 都是继承自EggApplication
class EggApplication extends EggCore {
    constructor(options = {}) {
        super(options);
        // 1.加载配置
        this.loader.loadConfig();

        // 2. app和agent之间通信的对象
        this.messenger = Messenger.create(this);

        // 监听app启动事件，开始didReady生命周期函数
        this.messenger.once('egg-ready', () => {
            this.lifecycle.triggerServerDidReady();
        });

        // 注册方法，把加载完的配置和各阶段的执行时间写入文件中，还有记录启动时间线，这些信息可以很方便的用于排查问题
        this.ready(() => process.nextTick(() => {
            const dumpStartTime = Date.now();
            this.dumpConfig();
            this.dumpTiming();
            this.coreLogger.info('[egg:core] dump config after ready, %s', ms(Date.now() - dumpStartTime));
        }));

        // 设置启动超时定时器，未在规定时间内启动(默认10分钟)，触发startTimeout事件
        this._setupTimeoutTimer();

        // 多进程研发模式增强 Leader/Follower 模式
        // 具体参考官方文档 https://eggjs.org/zh-cn/advanced/cluster-client.html
        this.cluster = (clientClass, options) => {

        });

        // 注册关闭方法
        this.beforeClose(async () => {
            // single process mode will close agent before app close
            if (this.type === 'application' && this.options.mode === 'single') {
                // 调用agent的close方法
                await this.agent.close();
            }

            for (const logger of this.loggers.values()) {
                logger.close();
            }

            // 调用通信实例关闭方法，移除一些监听器
            this.messenger.close();
            process.removeListener('unhandledRejection', this._unhandledRejectionHandler);
        });
    };

    this.BaseContextClass = BaseContextClass;
    this.Controller = BaseContextClass;
    this.Service = BaseContextClass;
    this.Subscription = BaseContextClass;
    this.BaseHookClass = BaseHookClass;
    this.Boot = BaseHookClass;
}
```



## 2.3 Agent 类和加载器

### 2.3.1 Agent 类

它是 `Egg` 中 `Agent` 进程默认实例化的类，它继承自公共的 `EggApplication` 类，核心逻辑为

1、实例化时会调用加载器的 `load()` 方法完成加载，这是 `Egg` 中核心的加载机制

2、还会调用 `dumpConfig()` 方法，把 `Egg` 的最终配置记录到文件中

* 因为`Egg` 的配置文件支持合并，把合并后最终起作用的配置记录到文件，当配置有问题时，有助于我们定位问题

3、实现了自己的加载器 `AgentWorkerLoader`，加载器对象是通过 `Symbol.for('egg#loader') ` 变量获取到



**Agent 源码如下**

> egg/lib/agent.js

```js
const EGG_LOADER = Symbol.for('egg#loader');

class Agent extends EggApplication {
    constructor(options = {}) {
        super(options);
        // 调用agent 自己实现的加载器的 load 方法，也是egg加载器中核心的加载逻辑
        this.loader.load(); 
      
        // 把配置记录到文件，可以查看egg内部配置合并后最终起作用的配置，也可用于排查问题
        this.dumpConfig();
    }
  
    // agent加载器，即 egg/lib/loader/agent_worker_loader.js
    get [EGG_LOADER]() {
        return AgentWorkerLoader;
    }
}
```



### 2.3.2 AgentWorkerLoader 加载器

`Egg` 加载器的基类是 `EggLoader` ，它根据文件加载的规则提供了一些内置的方法，它本身并不会去调用这些方法，而是由继承类调用。`AgentWorkerLoader` 就是继承了 `EggLoader`。



`AgentWorkerLoader` 是 `Agent` 自己实现的加载器，主要有两个核心的方法 `loadConfig()` 和 `load()`

1、`loadConfig`：主要用来加载插件和配置

* `loadPlugin`：加载所有插件
* `loadConfig`：加载配置

2、`load`：加载机制的核心逻辑

* `loadAgentExtend`：用于扩展 `Agent` 对象，会加载 `app/extend/agent.js` 的内容，然后挂载到 `agent` 应用对象上
* `loadContextExtend`：接着扩展 `Context` 对象，加载 `app/extend/context.js` 的内容，这些方法会挂载到 `context` 上下文对象上。因为 `Agent` 是一个 `koa` 实例，所以 `context` 也是 `koa` 的 `context` 对象，它是请求级别的对象
* `loadCustomAgent`：最后加载"启动自定义"，即加载每个加载单元根目录的 `agent.js` 文件中定义的生命周期函数，加载完成后，会开始生命周期的第一个阶段"配置文件即将加载阶段（`configWillLoad`）"



**AgentWorkerLoader 源码如下**

> egg/lib/loader/agent_worker_loader.js

```js
const EGG_LOADER = Symbol.for('egg#loader');

class AgentWorkerLoader extends EggLoader {
    /**
     * loadPlugin first, then loadConfig
     */
    loadConfig() {
        this.loadPlugin(); // 插件加载
        super.loadConfig(); // 配置加载
    }

    // agent的load方法的实现，以下各个方法的实现在egg-core/lib/loader/mixin/目录下
    load() {
        this.loadAgentExtend(); // 框架扩展，加载app/extend/agent.js到agent上
        this.loadContextExtend(); // 框架扩展，加载app/extend/context.js到context上

        this.loadCustomAgent();  // 加载启动自定义，会开始生命周期
    }
}
```



## 2.4 Application 类和加载器

### 2.4.1 Application 类

`Application` 是 `Egg` 中 `App` 进程默认实例化的类，它继承自公共的 `EggApplication` 类，其逻辑类似 `Agent` 类

1、调用加载器的 `load()` 方法完成加载，调用 `dumpConfig() `把配置记录到文件中

2、实现了自己的加载器 `AppWorkerLoader`，加载器对象也是通过 `Symbol.for('egg#loader') ` 变量获取到



**Application 源码如下**

> egg/lib/application.js

```js
// app加载
class Application extends EggApplication { 
    constructor(options = {}) {
        super(options);
        // 调用app加载的load方法
        this.loader.load();
      
        // 把配置写到文件中
        this.dumpConfig();
    }
  
    // application 对象的加载器，即 即 egg/lib/loader/app_worker_loader.js
    get [EGG_LOADER]() {
        return AppWorkerLoader;
    }
}
```



### 2.4.2 AppWorkerLoader 加载器

`AgentWorkerLoader` 是 `App` 进程实现的加载器，它也继承了 `EggLoader` 加载器基类。主要有两个核心的方法 `loadConfig()` 和 `load()`

1、`loadConfig`：主要用来加载插件和配置

* `loadPlugin`：加载所有插件
* `loadConfig`：加载配置

2、`load`：用来加载扩展对象、配置、约定的目录、路由等，核心逻辑如下

* `loadApplicationExtend`：用于扩展 `Application` 对象，会加载 `app/extend/application.js` 的内容，然后挂载到 `application` 应用对象上，`Application` 本质是一个 `koa` 全局对象

* `loadRequestExtend`：用于扩展 `Request` 对象，会加载 `app/extend/request.js `的内容，然后挂载到 `this.app.request` 对象上，它和 `koa` 的 `Request` 对象相同，是请求级别的对象

* `loadResponseExtend`：用于扩展 `Response` 对象，加载 `app/extend/response.js` 的内容，然后挂载到 `this.app.response` 对象上它和 `koa` 的 `Response` 对象相同，是请求级别的对象

* `loadContextExtend`：用于扩展 `Agent` 对象，加载 `app/extend/context.js` 的内容，这些方法会挂载到 `context` 上下文对象上。因为` Agent` 是一个 `koa` 实例，所以 `context` 也是 `koa` 的 `context` 对象，它是请求级别的对象

* `loadHelperExtend`：会扩展 `app.Helper` 对象，加载 `app/extend/helper.js` 的内容，挂载到 `this.app.Helper` 上，这个对象一般用来提供一些实用的 `utility` 函数

* `loadCustomLoader`：它可以通过配置把某个目录下的文件加载到 `app` 或着 `context` 对象上，这样我们不按 `egg` 约定到目录也可以加载自己目录到内容挂载到 `app` 和 `context`

  > 本质也是通过 `loadToContext` 挂载到 `context` 对象，通过 `loadToApp` 挂载到 `app` 对象
  
* `loadCustomApp`：加载“启动自定义”，即加载每个“加载单元”根目录的 `app.js` 文件中定义的生命周期函数，加载完成后，会开始生命周期的第一个阶段，即"配置文件即将加载阶段（`configWillLoad`）"

* `loadService`：加载 `app/service/` 目录下的文件，挂载到 `ctx.service` 对象上，这样就使得可以使用 `"ctx.service.文件名字.方法名"` 调用方法

* `loadMiddleware`：加载 `app/middleware/` 目录下的文件，挂载到 `app.middleware` 对象上

* `loadController`：加载 `app/controller/` 目录下的文件，挂载到 `app.controller` 对象上，使得可以使用 `"app.controller.文件名字.方法名" `调用方法

* `loadRouter`：导入 `app/router.js` 文件



**AppWorkerLoader 源码如下**

> egg/lib/loader/app_worker_loader.js

```js
// 继承自EggLoader加载器
class AppWorkerLoader extends EggLoader {
    /**
     * loadPlugin first, then loadConfig
     * @since 1.0.0
     */
    loadConfig() {
        this.loadPlugin(); // 插件加载
        super.loadConfig(); // 配置加载
    }

    /**
     * Load all directories in convention
     * @since 1.0.0
     */
    // 以下各个方法的实现在egg-core/lib/loader/mixin/目录下
    load() {
        // app > plugin > core
        this.loadApplicationExtend(); // 加载app/extend目录下的application.js，实际逻辑是loadExtend(name, proto)方法
        this.loadRequestExtend(); // 加载extend目录下的request.js
        this.loadResponseExtend(); // 加载extend目录下的response.js
        this.loadContextExtend(); // 加载extend目录下的context.js
        this.loadHelperExtend(); // 加载extend目录下的helper.js

        this.loadCustomLoader(); // 加载自定义加载器

        // app > plugin
        this.loadCustomApp(); // 加载自动自定义，会开始生命周期
        // app > plugin
        this.loadService();
        // app > plugin > core
        this.loadMiddleware();
        // app
        this.loadController();
        // app
        this.loadRouter(); // Dependent on controllers
    }
}
```

`AppWorkerLoader` 的父类是 `EggLoader` 加载器，父加载器定义了具体的加载规则

```js
// Egg-core/lib/loader/egg_loader.js
class EggLoader {
    // 获取加载单元
    getLoadUnits() {
        // 即按 插件->框架->应用 依次加载
    }
  
    loadToApp() {
      // 把文件方法挂载到app对象上
    }
  
    loadToContext() {
      // 把文件方法挂载到context对象上
    }
}

// 即egg中加载器各加载逻辑的主要实现，各实现在egg-core/lib/loader/mixin/目录下
const loaders = [
    require('./mixin/plugin'),
    require('./mixin/config'),
    require('./mixin/extend'),
    require('./mixin/custom'),
    require('./mixin/service'),
    require('./mixin/middleware'),
    require('./mixin/controller'),
    require('./mixin/router'),
    require('./mixin/custom_loader'),
];

for (const loader of loaders) {
    Object.assign(EggLoader.prototype, loader);
}
```



# 3 加载机制前置知识

在讲解加载机制的具体实现之前，先来了解下 `Egg` 中的框架、插件、和一些加载器的 `API`，因为加载机制会自动加载所有框架和插件



## 3.1 框架 framework

### 3.1.1 什么是框架

`Egg` 为我们提供了框架（`framework`）定制的能力，我们可以基于 `Egg` 去封装上层框架，进而添加自己的功能，并且 `Egg` 是支持多层继承的，`Egg` 加载器加载框架的顺序是从父框架一直到子框架

1、这里的 **"基于 `Egg`"** 可以简单理解为是基于 `Egg` 内置的 `Agent` 类和 `Application` 类，所以我们自己定制一个框架时需要实现这两个类，并实现它们的方法，比如 `EGG_PATH` 方法是必须实现的，它表示当前框架所在的目录，即我们定制的框架的目录，这样 `Egg` 的 `Loader` 加载器才会从这个路径读到我们这个框架

2、我们实现了 `Application` 和 `Agent` 的子类，需要分别导出为 `Application` 类和 `Agent` 类 ，因为 `Agent Worker` 和 `Application Worker` 进程默认是实例化框架 `framework` 路径下的 `Agent` 和 `Application` 类

3、多层继承是指 `Egg` 的 `Application` 和 `Agent` 可以有子类，且子类又可以有子类，比如我们有一个企业框架继承自 `Egg`，又有一个部门框架继承自企业框架



### 3.1.2 定制一个框架

例如定制一个企业的框架的流程如下

1、 实现 `egg.Application` 和 `egg.Agent`，覆写 `EGG_PATH` 方法，其实它是 `Symbol.for('egg#eggPath')` 变量，代码如下

```js
// lib/framework.js
// 必须是Symbol.for('egg#eggPath') 指定当前框架的路径，加载器会搜索此变量的值作为框架的路径，加载器就能读到此框架
const EGG_PATH = Symbol.for('egg#eggPath');

const egg = require('egg');
// 定义一个企业框架，继承 Egg 的 Application
class EnterpriseApplication extends egg.Application {
  get [EGG_PATH]() {
    // 返回框架的路径
    return path.dirname(__dirname);
  }
}

// 实现 Agent
class EnterpriseAgent extends egg.Agent {
  get [EGG_PATH]() {
    return path.dirname(__dirname);
  }
}

// 覆盖 Egg 默认的 Application，即把自己实现的类导出为 Application 类
module.exports = Object.assign(egg, {
  Application: EnterpriseApplication,
  Agent: EnterpriseAgent,
});
```

2、指定框架名（在 `package.json` 中指定 `egg.framework`，默认值是 `egg`）

> `Loader` 会从 `node_modules` 中寻找指定模块作为框架，并载入其导出的 `Application`

```js
// package.json 
{
  "name": "enterprise",
  "dependencies": {
    "egg": "^2.0.0"
  }
  "scripts": {
    "dev": "egg-bin dev"
  },
  "egg": {
    "framework": "enterprise" // 指定框架名为enterprise
  }
}
```

3、在 `index.js` 导出 `Application` 和 `Agent`，如

```js
module.exports = require('./lib/framework.js');
```



因为 `Egg` 支持多层框架，如果要再实现一个部门框架，此时是三层框架：部门框架（`department`）-> 企业框架（`enterprise`）-> `Egg`，部门框架核心逻辑如下

```js
// lib/framework.js
const EnterpriseApplication = require('enterprise').Application;
const EnterpriseAgent = require('enterprise').Agent;

// 继承自己企业框架的 EnterpriseApplication
class DepartmentApplication extends EnterpriseApplication {
  get [EGG_PATH]() {
    return path.dirname(__dirname);
  }
}

// 实现 EnterpriseAgent
class DepartmenAgent extends EnterpriseAgent {
  get [EGG_PATH]() {
    return path.dirname(__dirname);
  }
}

module.exports = Object.assign(egg, {
  Application: DepartmentApplication,
  Agent: DepartmenAgent,
});
```



### **3.1.3 自定义 Loader 加载器**

`Egg` 约定了使用 `Symbol.for('egg#loader')` 变量来自定义 `Loader` 加载器，主要原因还是为了使用原型链。上层框架覆盖  `Symbol.for('egg#loader')`  变量，即可覆盖底层的加载器，`Agent` 和 `Application`  都可以定义自己的加载器，目前 `Egg` 默认的就是 `AgentWorkerLoader` 和 `AppWorkerLoader` 加载器



**例如为上面的企业框架自定义 Application 的加载器**

1、要实现自己的加载器，先继承默认的 `Application` 加载器，即 `egg.AppWorkerLoader` 

2、覆盖 `Egg` 的 `Loader`，即   `Symbol.for('egg#loader')`  变量，`Egg` 启动时会通过 `EGG_LOADER` 方法获取这个自己实现的 `Loader`

3、把加载器导出为 `Egg` 的 `AppWorkerLoader` 对象，以便上层框架进行扩展

```js
// lib/framework.js
const path = require('path');
const egg = require('egg');
const EGG_PATH = Symbol.for('egg#eggPath');

// 1、实现 AppWorkerLoader
class EnterpriseLoader extends egg.AppWorkerLoader {
  load() {
    super.load();
    // 进行自己的扩展
  }
}

class EnterpriseApplication extends egg.Application {
  get [EGG_PATH]() {
    // 返回 framework 路径
    return path.dirname(__dirname);
  }
  
  // 2、覆盖 Egg 的 Loader，启动时将使用这个 Loader
  get [EGG_LOADER]() {
    return EnterpriseLoader;
  }
}

// 覆盖了 Egg 的 Application
module.exports = Object.assign(egg, {
  Application: EnterpriseApplication,
  // 自定义的 Loader 也需要导出，以便上层框架进行扩展
  AppWorkerLoader: EnterpriseLoader,
});
```

> `Agent` 要自定义加载器时也类似 `Application` 



**基于框架启动**

Q：定制完一个框架后，如果这是一个最上层的应用框架，要怎么使用框架呢？

A：以 `egg-sciprts` 启动程序为例，它用的是 `startCluster` 启动 `Master`，可以传入 `baseDir` 和 `framework`，`framework` 默认就是 `egg` 包的路径，如果我们传入自己的 `framework` 路径，即传入自己的框架的路径，那 `Egg` 的 `Application` 和 `Agent` 进程启动时就会使用自己这个 `framework` 路径的 `Application` 和 `Agent` 去实例化 `Application` 和 `Agent` 对象。



## 3.2 插件 plugin

### **3.2.1 目录结构**

插件的目录结构，和"应用（`app`）"几乎一样，区别如下 

1、插件没有 `router` 和 `controller`，因为

* 路由一般和应用强绑定的，不具备通用性
* 一个应用可能依赖很多个插件，如果插件支持路由可能会导致路由冲突
* 如果确实有统一路由的需求，可以考虑在插件里通过中间件来实现

2、插件需要在 `package.json` 中的 `eggPlugin` 节点指定插件特有的信息，如

```js
{
  "name": "egg-rpc",
  "eggPlugin": {
    "name": "rpc", // 插件名（必须配置），具有唯一性，配置依赖关系时会指定依赖插件的 name
    "dependencies": ["registry"], // 是一个数组，当前插件强依赖的插件列表（如果依赖的插件没找到，应用启动失败）
    "optionalDependencies": ["vip"], // 是一个数组，当前插件的可选依赖插件列表（如果依赖的插件未开启，只会 warning，不会影响应用启动）
    "env": ["local", "test", "unittest", "prod"] // 是一个数组，只有在指定运行环境才能开启
  }
}
```

3、插件没有 `plugin.js`

* `eggPlugin.dependencies` 只是用于声明依赖关系，而不是引入插件或开启插件
* 如果期望统一管理多个插件的开启和配置，上层框架处理



注意：插件是自己管理依赖的，应用在加载所有插件前会预先从它们的 `package.json` 中读取 `eggPlugin.dependencies` 和 `eggPlugin.optionalDependencies` 节点，然后根据依赖关系计算出加载顺序。`dependencies` 和 `optionalDependencies` 的取值是另一个插件的 `eggPlugin.name`，而不是 `package name`。



### **3.2.2 插件的作用**

1、扩展内置对象(`app/extend/` 目录)

- `app/extend/request.js` - 扩展 `Koa#Request` 类
- `app/extend/response.js` - 扩展 `Koa#Response` 类
- `app/extend/context.js` - 扩展 `Koa#Context` 类
- `app/extend/helper.js` - 扩展 `Helper` 类
- `app/extend/application.js` - 扩展 `Application` 类
- `app/extend/agent.js` - 扩展 `Agent` 类

2、插入自定义中间件，例如

```js
// 将自定义的static 中间件放到 bodyParser 之前
const index = app.config.coreMiddleware.indexOf('bodyParser');
app.config.coreMiddleware.splice(index, 0, 'static');
```

3、在应用启动时做一些初始化工作

* 可以 `app.js` 或 `agent.js` 中做一些工作，如有异步启动逻辑，可以使用 `app.beforeStart` 方法（新版本不建议使用，推荐使用 `didLoad`）

```js
// ${plugin_root}/app.js
const MyClient = require('my-client');

module.exports = (app) => {
  app.myClient = new MyClient();
  app.myClient.on('error', (err) => {
    app.coreLogger.error(err);
  });
  app.beforeStart(async () => {
    await app.myClient.ready();
    app.coreLogger.info('My client is ready');
  });
};
```

4、设置定时任务

* 在 `package.json` 里设置依赖 `schedule` 插件
* 在 `${plugin_root}/app/schedule/` 目录下新建文件，编写你的定时任务

```js
// package.json
{
  "name": "your-plugin",
  "eggPlugin": {
    "name": "your-plugin",
    "dependencies": ["schedule"]
  }
}

// ${plugin_root}/app/schedule/
exports.schedule = {
  type: 'worker',
  cron: '0 0 3 * * *',
  // interval: '1h',
  // immediate: true
};

exports.task = async (ctx) => {
  // 你的逻辑代码
};
```



## 3.3 加载单元

`Egg` 内部有一个加载单元的概念，所有的插件、框架 和 应用，每一个个体都称为 `Egg` 的加载单元。



在加载器的加载过程中，会获取所有的加载单元，通过 `getLoadUnits()` 方法实现的，其核心逻辑为

1、先加载插件，从 `orderPlugins` 变量中获取所有插件的信息，这个 `orderPlugins` 变量存放了所有 `Egg` 的插件信息，是在 `loadConfig` 阶段处理的（下面会讲解）

2、接着加载框架，从 `eggPaths` 变量获取所有框架的信息，它存放了所有 `Egg` 的框架的信息，即我们自定义的框架，如果是多层框架，也会存放这多层框架的信息，它是通过 `getEggPaths()` 方法处理的

3、最后加载应用， `App Worker` 进程下就是 `Application` 类，`Agent` 进程下就是 `Agent` 类，可以统称为 `app`，它的路径时启动时传入的 `baseDir` 参数



**源码如下**

```js
// egg-core/lib/loader/egg_loader.js
  getLoadUnits() {
    if (this.dirs) {
      return this.dirs;
    }

    const dirs = this.dirs = [];

    if (this.orderPlugins) {
      for (const plugin of this.orderPlugins) {
        dirs.push({
          path: plugin.path,
          type: 'plugin',
        });
      }
    }

    // framework or egg path
    for (const eggPath of this.eggPaths) {
      dirs.push({
        path: eggPath,
        type: 'framework',
      });
    }

    // application
    dirs.push({
      path: this.options.baseDir,
      type: 'app',
    });

    debug('Loaded dirs %j', dirs);
    return dirs;
  }
```



## 3.4 加载器基础 API

加载器提供了一些基础的 `API`，方便在扩展时简化代码



### 3.4.1 loadToApp

`loadToApp()` 的作用是将一个目录下的文件挂载到 `app` 对象上

* 假设加载 `controller` 控制器，此时加载的是 `controller` 目录下的文件，加载完成可以通过文件名就能从 `app` 对象中访问到对应文件的内容，例如 `app/controller/home.js` 会被加载到 `app.controller.home`，可以通过参数控制文件名的命名规则，比如驼峰命名



`loadToApp(directory, property, LoaderOptions) `有三个参数：

1. `directory` 可以是字符串或数组，`Loader `会从这些目录中加载文件
2. `property` 是 `app` 的属性名
3. `LoaderOptions`包含了一些配置选项，具体配置项可以参考[这里](https://www.eggjs.org/zh-CN/advanced/loader#loaderoptions)



### 3.4.2 loadToContext

`loadToContext()` 的作用是将文件加载到 `ctx` 上（而不是 `app`），并且支持懒加载。加载操作会将文件放到一个临时对象中，在调用 `ctx API` 时才去实例化。

> 例如加载  `service` 目录下的文件时就使用到了 `loadToContext`

例如

```js
// 以下为示例，请使用 loadService
// app/service/user.js
const Service = require('egg').Service;
class UserService extends Service {}
module.exports = UserService;

// app.js
// 获取所有的 loadUnit
const servicePaths = app.loader
  .getLoadUnits()
  .map((unit) => path.join(unit.path, 'app/service'));

app.loader.loadToContext(servicePaths, 'service', {
  // service 需要继承 app.Service，因此需要 app 参数
  // 设置 call 为 true，会在加载时调用函数，并返回 UserService
  call: true,
  // 将文件加载到 app.serviceClasses
  fieldClass: 'serviceClasses',
});
```

文件加载完成后，`app.serviceClasses.user` 就代表 `UserService` 类。当调用 `ctx.service.user` 时，会实例化 `UserService` 类。因此，这个类只有在每次请求中首次被访问时才会实例化。实例化后，对象会被缓存，同一个请求中多次调用也只实例化一次。



### 3.4.3 loadFile

此函数用来加载文件，比如加载路由 `app/router.js`  和加载 `config` 配置文件就使用了 `loadFile`。如果文件是导出了一个函数，那这个函数会被调用，`app` 作为参数传入；如果不是函数，则直接使用文件导出的值。例如

```js
// app/xx.js
module.exports = (app) => {
  console.log(app.config);
};

// app.js
// 以 app/xx.js 为例子，在 app.js 中加载此文件：
const path = require('path');
module.exports = (app) => {
  // 加载app/xx.js文件
  app.loader.loadFile(path.join(app.config.baseDir, 'app/xx.js'));
};
```



# 4 加载器各方法源码分析

前面了解了 `Egg` 的 `AgentWorkerLoader` 和 `AppWorkerLoader` 都有 `loadConfig` 和 `load` 方法，这两个加载器的 `loadConfig` 方法实现一样的，而 `load` 方法 `Agent` 和 `Application` 有各自的实现，下面通过源码分析下各个加载方法的实现



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/egg%E5%8A%A0%E8%BD%BD%E5%99%A8.png)



## 4.1 加载插件 plugin

**加载插件 loadPlugin 的流程如下**

1、先获取所有要加载的目录，包含以下目录

* `baseDir` ：当前应用的路径
* 所有 `Egg` 的框架路径
* `cwd`：当前路径

2、读取所有插件配置 

（1）读取所有`"框架 framework"`用到的插件：`loadEggPlugins`

* 注意：如果存在多个有继承关系的框架，会优先加载父框架，一直加载到子类的框架
* 每个框架内的插件配置文件，是根据 `plugin.default.js -> plugin.${scope}.js -> plugin.${env}.js -> plugin.${scope}_${env}.js` 的文件顺序读取
* 如果多层框架配置了相同的插件，则这个插件的配置会进行合并，如果有相同的属性，则会进行覆盖（如果后面框架的属性是空数组，则不覆盖前面已有的值）

（2）读取`"当前应用（app）"`的配置： `loadAppPlugins`

* 这里的 `app` 指启动时传入的 `baseDir` 目录参数，默认当前项目路径，配置文件的加载规则也是跟上面框架配置文件的加载规则一样

（3）加载自定义配置：`loadCustomPlugins`

*  会先从 `process.env.EGG_PLUGINS` 环境变量中加载，它是一个 `json` 字符串

* 如果启动时有传 `plugins` 参数，那这个参数会覆盖掉 `process.env.EGG_PLUGINS`（使用 `Object.assign` 覆盖），这里 `plugins` 是对象格式，如

  ```js
// plugins格式
  {
    mysql = {
      enable: true,
       package: 'egg-mysql',
    }
    // ...... 其他插件
  }
  ```

3、合并所有插件的配置，按 `框架插件-> 应用(app)插件 -> 自定义插件` 的顺序合并，相同属性则覆盖

4、接着整理所有插件 `plugin` 的信息，主要逻辑如下

（1）获取每个插件的路径：优先使用插件中配置的 `path` 属性，它是一个绝对路径；如果没有 `path` 就用 `package` 属性，如果还是没有 `package` 属性就用 `name` 属性（`package` 和 `name` 配置的是模块名），知道会找到插件的路径

（2）从插件路径下的 `package.json` 读取 `eggPlugin` 属性，插件必须有 `eggPlugin` 属性，它配置了插件的信息，主要是读取里面的 `dependencies`、`optionalDependencies`、`env`，然后挂载 到当前 `plugin` 对象中

（3）判断插件配置的 `env` 环境参数是否符合当前环境，不符合则插件设置 `enable` 为 `false`，表示禁用该插件

5、接着把处理过后的所有插件对象再排个序，主要是解决所有插件相互的依赖关系

>   根据依赖等信息，递归调用 `sequence` 函数获得插件的加载顺序（会处理循环依赖，依赖缺失等问题）

6、最后把插件挂载在加载器的 `plugins` 属性上



**源码分析**

> loadPlugin 的源码位于 egg-core/lib/loader/mixin/plugin.js

```js
loadPlugin() {

    // 查找所有目录
    this.lookupDirs = this.getLookupDirs();
    this.allPlugins = {};
    // 读取所有“framework框架”所配置的插件
    this.eggPlugins = this.loadEggPlugins();
    // 读取app应用的配置
    this.appPlugins = this.loadAppPlugins();
    // 读取自定义配置
    this.customPlugins = this.loadCustomPlugins();

    // 合并所有配置
    this._extendPlugins(this.allPlugins, this.eggPlugins);
    this._extendPlugins(this.allPlugins, this.appPlugins);
    this._extendPlugins(this.allPlugins, this.customPlugins);

    const enabledPluginNames = []; // enabled plugins that configured explicitly
    const plugins = {};
    const env = this.serverEnv;
    for (const name in this.allPlugins) {
      const plugin = this.allPlugins[name];

      // 读取每个插件的路径
      // resolve the real plugin.path based on plugin or package
      plugin.path = this.getPluginPath(plugin, this.options.baseDir);

      // 从插件路径下的package.json读取eggPlugin属性，即读插件的信息
      // read plugin information from ${plugin.path}/package.json
      this.mergePluginConfig(plugin);

      // 判断插件配置的env环境参数是否符合当前环境
      // disable the plugin that not match the serverEnv
      if (env && plugin.env.length && !plugin.env.includes(env)) {
        this.options.logger.info('Plugin %s is disabled by env unmatched, require env(%s) but got env is %s', name, plugin.env, env);
        plugin.enable = false;
        continue;
      }

      plugins[name] = plugin;
      if (plugin.enable) {
        enabledPluginNames.push(name);
      }
    }

    // 插件排序，解决包的依赖关系
    // retrieve the ordered plugins
    this.orderPlugins = this.getOrderPlugins(plugins, enabledPluginNames, this.appPlugins);

    const enablePlugins = {};
    for (const plugin of this.orderPlugins) {
      enablePlugins[plugin.name] = plugin;
    }

    // 把处理完的插件赋予plugins
    /**
     * Retrieve enabled plugins
     * @member {Object} EggLoader#plugins
     * @since 1.0.0
     */
    this.plugins = enablePlugins;

    this.timing.end('Load Plugin');
  },
    
```



## 4.2 加载配置 config

**加载配置 loadConfig 的流程如下**

1、加载`"当前应用（app）"`的配置

* 先加载 `config.default` 文件，再加载 `config.${this.serverEnv}` 文件，同名属性会合并覆盖（是深度克隆，通过 [extend2](https://github.com/eggjs/extend2) 库实现的）

2、加载所有"加载单元"的配置 ，先按文件再按"加载单元"的顺序

* 获取所有约定好命名的配置文件，遍历每种配置文件，文件是按 `config.default -> config.{scope} -> config.{env} -> config.{scope}_{env}` 的顺序
* 针对每种配置文件，都会获取所有“加载单元”下的该种配置文件，如果有同名属性的配置，会进行合并覆盖（也是用的 `extend2` 库实现）

```js
// 配置加载顺序伪代码如下
for (const filename of 所有配置文件名字) {
  for (const loadUnit of 所有加载单元) {
     // 加载配置后合并
  }
}

// 加载顺序如下
// plugin config.default
// framework config.default
// app config.default
// 
// plugin config.{scope}
// framework config.{scope}
// app config.{scope}
// 
// plugin config.{env}
// framework config.{env}
// app config.{env}
```

3、加载环境变量的配置 `process.env.EGG_APP_CONFIG`，它是一个 `json` 字符串，加载完后如果有同名属性也会覆盖之前的配置

4、处理中间件配置，加载中间件的配置到 `config` 的 `appMiddleware` 和 `coreMiddleware` 属性中

* 一般 `app.config.appMiddleware`  表示应用层定义的中间件，`app.config.coreMiddleware` 表示框架默认中间件，它们在中间件加载阶段都会被加载，并挂载到 `app.middleware` 上

* 我们的`"应用（app）"`可以在 `config.default.js` 中配置 `middleware` 中间件数组，它会被挂载到 `app.config.appMiddleware` 中。而框架和插件不支持在 `config.default.js` 中配置  `middleware`，可以在 `app.js` 中使用`app.config.coreMiddleware.unshift('中间件名')`的方式使用插入中间件

5、把配置挂载到 `config` 对象中



**源码分析**

> 源码位于 egg-core/lib/loader/mixin/config.js

```js
loadConfig() {
    // 1. 加载当前 application 项目的配置
    // -> 先加载config.default，后加载config.${this.serverEnv}的，出现同名属性，后面的会直接覆盖掉前面的(具体实现为extend2包)
    const appConfig = this._preloadAppConfig();

    // 2. 各配置文件加载顺序
    //   plugin config.default
    //     framework config.default
    //       app config.default
    //         plugin config.{env}
    //           framework config.{env}
    //             app config.{env}
  
    // 先读取所有配置文件的命名，然后遍历每个配置文件并按"加载单元"的顺序依次加载
    // ->config.default.js  
    //   config.${serverScope}.js 
    //   config.${serverEnv}.js 
    //   config.${serverScope}_${serverEnv}.js文件下的插
    for (const filename of this.getTypeFiles('config')) {
        // 获取所有加载单元，按 "插件-> 框架 -> 应用" 的顺序获取
        for (const unit of this.getLoadUnits()) {
            const config = this._loadConfig(unit.path, filename, isApp ? undefined : appConfig, unit.type);
            // 深度克隆，同名属性会覆盖，数组也会覆盖
            extend(true, target, config);
        }
    }

     // 3. 加载环境process.env.EGG_APP_CONFIG的配置
     const envConfig = this._loadConfigFromEnv();
     extend(true, target, envConfig);

    // 4. 加载中间件的配置到coreMiddleware属性(框架默认中间件)和appMiddleware属性(应用层定义的中间件)
    target.coreMiddleware = target.coreMiddlewares = target.coreMiddleware || [];
    target.appMiddleware = target.appMiddlewares = target.middleware || [];

    // 5. 挂载到config属性
    this.config = target;
}
```



## 4.3 加载扩展文件 extend

`Egg` 可以定义多种扩展文件，加载扩展文件就是加载某个文件的内容到某个原型对象上。各种加载扩展文件到核心方法都是 `loadExtend()`。



例如 `loadAgentExtend()`  其实就是调用了 `this.loadExtend('agent', this.app)` 方法，表示加载 `app/extend/agent.js` 扩展文件的内容，然后挂载到 `this.app` 对象上，`loadAgentExtend` 的逻辑如下

1、先获取所有扩展文件的路径

* 会获取所有“加载单元"的 `app/extend/agent.js` 扩展文件

2、遍历每个 `agent.js` 扩展文件

* 导入文件
* 获取所有属性（支持 `Symbol` 属性名）
* 然后把属性定义到 `this.app` 对象上



**各扩展文件的源码如下**

> 源码位于 egg-core/lib/loader/mixin/extend.js

```js
// 加载app/extend/agent.js到 'app的原型'上
loadAgentExtend() {
    this.loadExtend('agent', this.app);
},

// 加载app/extend/application.js到'app的原型'上
loadApplicationExtend() {
    this.loadExtend('application', this.app);
},

// 加载app/extend/request.js到app.request的原型上 
loadRequestExtend() {
    this.loadExtend('request', this.app.request);
},

// 加载app/extend/response.js到app.response的原型上 
loadResponseExtend() {
    this.loadExtend('response', this.app.response);
},

// 加载app/extend/context.js到app.context的原型上 
loadContextExtend() {
    this.loadExtend('context', this.app.context);
},

// 加载app/extend/helper.js到app.Helper的原型上 
loadHelperExtend() {
    if (this.app && this.app.Helper) {
        this.loadExtend('helper', this.app.Helper.prototype);
    }
},


// 实际的加载扩展逻辑   

// 加载app/extend/xx.js 到 prototype上
// ->获取所有扩展文件然后加载内容到对应原型上，即获取所有加载单元(插件-> 框架 -> 应用)的app/extend的文件
loadExtend(name, proto) {
    const filepaths = this.getExtendFilePaths(name);
      ......
},

getExtendFilePaths(name) {
    return this.getLoadUnits().map(unit => path.join(unit.path, 'app/extend', name));
},
```



## 4.4 启动自定义 custom

启动自定义即启动生命周期函数，`App` 进程的具体实现是 `loadCustomApp()` 方法，`Agent` 进程的具体实现是 `loadCustomAgent()` 方法，更详细的解析请看下面“生命周期”章节



## 4.5  加载中间件 middleware

**加载中间件 loadMiddleware 的流程如下**

1、获取所有"加载单元" ，每个都获取 `app/middleware`  目录的文件，读取文件的内容挂载到 `app` 的 `middlewares` 属性中

2、接着把中间件定义到 `app.middleware` 属性上

4、读取我们配置的所有中间件名字，优先 `app.config.coreMiddleware` 的配置，然后再是 `app.config.appMiddleware` 中的配置，然后逐个检查

* 检查是否存在对应中间件文件，不存在则报错
* 检查中间件是否重名，重名则会报错（中间件的名字就是 `middleware` 目录下的文件名（会把文件名使用`toLowerCase` 转换）
* 最后调用 `app.use` 应用中间件



**源码分析**

> 源码位于 egg-core/lib/loader/mixin/middleware.js

```js
 // 1. 获取所有加载单元的app/middleware目录   插件->框架 ->应用
loadMiddleware(opt) {
    // load middleware to app.middleware
    opt = Object.assign({
        call: false,
        override: true,
        caseStyle: 'lower',
        directory: this.getLoadUnits().map(unit => join(unit.path, 'app/middleware')),
    }, opt);

    const middlewarePaths = opt.directory;
    // 2.将middlewares目录中的中间件读取到app的middlewares属性中
    this.loadToApp(middlewarePaths, 'middlewares', opt);

    // 3.遍历中间件定义到app.middleware属性上
    // 所以能够通过app.middleware.xxx来调用名为的xxx中间件
    for (const name in app.middlewares) {
        Object.defineProperty(app.middleware, name, {
            get() {
                return app.middlewares[name];
            },
            enumerable: false,
            configurable: false,
        });
    }

    // 4.根据配置文件配置的中间件加载中间件
    // -> 加载顺序是由app.config.coreMiddleware和app.config.appMiddleware这两个数组拼接而成的
    // -> 如果配置中配置了错误的中间件，会报错
    // -> 如果配置了相同名字的中间件，会报错
    const middlewareNames = this.config.coreMiddleware.concat(this.config.appMiddleware);
    for (const name of middlewareNames) {
        // 配置中配置了错误的中间件，会报错
        if (!app.middlewares[name]) {
            throw new TypeError(`Middleware ${name} not found`);
        }
        // 如果配置了相同名字的中间件，会报错
        if (middlewaresMap.has(name)) {
            throw new TypeError(`Middleware ${name} redefined`);
        }
        let mw = app.middlewares[name];
        mw = mw(options, app);

        // 对中间件函数的包裹处理，如enable/match/ignore等配置项和generator函数的处理
        mw = wrapMiddleware(mw, options);
        if (mw) {
            // 5.加载中间件
            app.use(mw);   
            ...
        }
    }
}

```

## 4.6 加载 service

**加载 service（loadService） 流程如下**

1、获取所有“加载单元” ，获取每个加载单元的 `app/service`  目录下的文件

2、读取 `service` 目录下文件的内容挂载到 `app.context` 的 `service` 属性中（使用了 `loadToContext()`  方法），所以可以使用 `ctx.service.xxx` 调用



**源码如下**

> egg-core/lib/loader/mixin/service.js

```js
loadService(opt) {
    // 依然还是按加载单元的顺序加载app/service目录下的所有文件
    opt = Object.assign({
        call: true,
        caseStyle: 'lower',
        fieldClass: 'serviceClasses',
        directory: this.getLoadUnits().map(unit => path.join(unit.path, 'app/service')),
    }, opt);
    const servicePaths = opt.directory;

    // 把所有service挂载到context对象上
    this.loadToContext(servicePaths, 'service', opt);
}
  
```

## 4.7 加载 controller

**加载控制器 loadController 流程如下**

1、获取 `"应用（app）"`，即 `baseDir` 路径下 `app/controller`  目录下的文件

* 注意这里不再是获取加载单元，所以插件和框架没有 `controller`

2、读取 `controller` 目录下文件的内容挂载到 `app.context` 的 `controller` 属性中（也使用了 `loadToContext()` 方法），所以可以使用 `ctx.controller.xxx` 调用



**源码如下**

> egg-core/lib/loader/mixin/controller.js

```js
loadController(opt) {
    opt = Object.assign({
        caseStyle: 'lower',
        directory: path.join(this.options.baseDir, 'app/controller'),
        // 初始化函数
        initializer: (obj, opt) => {
            // return class if it exports a function
            // ```js
            // module.exports = app => {
            //   return class HomeController extends app.Controller {};
            // }
            // ```
            if (is.function(obj) && !is.generatorFunction(obj) && !is.class(obj) && !is.asyncFunction(obj)) {
                obj = obj(this.app);
            }
            // 如果类，用wrapClass函数包裹
            if (is.class(obj)) {
                obj.prototype.pathName = opt.pathName;
                obj.prototype.fullPath = opt.path;
                return wrapClass(obj);
            }
            // 如果是对象，如果对象属性上函数，则用generator包裹
            if (is.object(obj)) {
                return wrapObject(obj, opt.path);
            }
            // 如果是generator函数，手动包裹成对象
            // support generatorFunction for forward compatbility
            if (is.generatorFunction(obj) || is.asyncFunction(obj)) {
                return wrapObject({ 'module.exports': obj }, opt.path)['module.exports'];
            }
            return obj;
        },
    }, opt);
    const controllerBase = opt.directory;

    // 把controller挂载到app上，即app.controller.xxx
    this.loadToApp(controllerBase, 'controller', opt);
}
```

## 4.8 加载路由 router

**加载路由 loadRouter 流程如下**

导入 `"app 应用目录"`（即 `baseDir` ）下的 `app/router.js` 文件

> 插件和框架没有路由



源码如下

> egg-core/lib/loader/mixin/router.js

```js
loadRouter() {
    // 加载 router.js  在app下的router文件中定义，所以直接读取代码即可
    this.loadFile(path.join(this.options.baseDir, 'app/router'));
}
```

## 4.9 自定义加载 custom_loader

自定义加载是由 `loadCustomLoader()` 方法实现的，它可以让我们通过配置的方式去加载对应目录下的文件，然后挂载到 `app` 或 `context` 对象上。这样我们不按 `egg` 约定的目录也可以加载自己目录到内容挂载到 `app` 和 `context`，本质也是通过 `loadToContext()` 挂载到 `context` 对象，通过 `loadToApp()` 挂载到 `app` 对象，具体的配置方法可参考[egg官方文档](https://www.eggjs.org/zh-CN/advanced/loader#customloader)



**源码如下**

> egg-core/lib/loader/mixin/custom_loader.js

```js
loadCustomLoader() {
    const loader = this;
    const customLoader = loader.config.customLoader || {};

    for (const property of Object.keys(customLoader)) {
      const loaderConfig = Object.assign({}, customLoader[property]);
      assert(loaderConfig.directory, `directory is required for config.customLoader.${property}`);

      let directory;
      if (loaderConfig.loadunit === true) {
        directory = this.getLoadUnits().map(unit => path.join(unit.path, loaderConfig.directory));
      } else {
        directory = path.join(loader.appInfo.baseDir, loaderConfig.directory);
      }
      // don't override directory
      delete loaderConfig.directory;

      const inject = loaderConfig.inject || 'app';
      // don't override inject
      delete loaderConfig.inject;

      switch (inject) {
        case 'ctx': {
          assert(!(property in loader.app.context), `customLoader should not override ctx.${property}`);
          const defaultConfig = {
            caseStyle: 'lower',
            fieldClass: `${property}Classes`,
          };
          // 挂载到 context 对象上
          loader.loadToContext(directory, property, Object.assign(defaultConfig, loaderConfig));
          break;
        }
        case 'app': {
          assert(!(property in loader.app), `customLoader should not override app.${property}`);
          const defaultConfig = {
            caseStyle: 'lower',
            initializer(Clz) {
              return is.class(Clz) ? new Clz(loader.app) : Clz;
            },
          };
          // 挂载到 app 对象上
          loader.loadToApp(directory, property, Object.assign(defaultConfig, loaderConfig));
          break;
        }
        default:
          throw new Error('inject only support app or ctx');
      }
    }
  },
```



# 5 生命周期函数

`Egg` 支持启动自定义 ，提供了统一的入口文件（`app.js` 或 `agent.js`）进行启动过程自定义，其实就是生命周期函数。以 `App` 进程为例，在 `App` 加载过程中调用了 `loadCustomApp() ` 方法，这个方法的作用就是处理 `App` 进程的生命周期。

> `agent` 进程是通过 `loadCustomAgent()` 方法加载 `agent.js` 文件处理，流程一样

所有的生命周期函数如下图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E5%87%BD%E6%95%B0.png)



下面以 `App` 进程的生命周期处理为例，即 `loadCustomApp()` 方法的处理，讲解下生命周期的每个阶段的逻辑

## 5.1 configWillLoad

生命周期的第一个阶段的触发是在 `loadCustomApp()` 方法中触发的，`loadCustomApp()` 的流程如下

1、先加载所有"加载单元"根目录下的 `app.js` 文件

* 会把整个 `app.js` 文件的内容存到数组里（新版是用 `class` 定义的类；旧版是用 `function` 定义，旧版会被包装为 `class` 的形式，然后放到 `configDidLoad` 阶段里执行）

2、加载完所有 `app.js` 后，会实例化每个类，构造方法会传入当前的 `app` 参数，最后把这些实例放到 `BOOTS` 数组里（后面的阶段都会从该数组里取每个对象的生命周期方法去执行）

3、接着调用 `triggerConfigWillLoad()` 触发生命周期的第一个阶段（`configWillLoad` 阶段），即配置文件即将加载阶段（这是最后动态修改配置的时机，它是一个同步函数，此时配置已经加载，但还并未生效，这是应用层修改配置的最后机会）

* 它会遍历 `BOOTS` 数组，依次判断每个对象是否存在 `configWillLoad` 方法，存在则调用该方法

4、最后接着触发生命周期的下一个阶段



**源码如下**

```js
// egg-core/lib/loader/mixin/custom.js 
// 加载`所有加载单元`根目录下的app.js定义的周期函数，并触发'配置文件即将加载configWillLoad'阶段
loadCustomApp() {
    this[LOAD_BOOT_HOOK]('app');
  
    // 触发'配置文件即将加载configWillLoad'阶段，开始生命周期
    this.lifecycle.triggerConfigWillLoad();
},


// 加载`所有加载单元`下的app.js，然后实例化
[LOAD_BOOT_HOOK](fileName) {
    for (const unit of this.getLoadUnits()) {
        const bootFilePath = this.resolveModule(path.join(unit.path, fileName));
        const bootHook = this.requireFile(bootFilePath);
            ......
        if (is.class(bootHook)) {
            bootHook.prototype.fullPath = bootFilePath;
            // if is boot class, add to lifecycle
            this.lifecycle.addBootHook(bootHook);
        } else if (is.function(bootHook)) {
            // 不是类的形式，要包装成类的形式
            this.lifecycle.addFunctionAsBootHook(bootHook);
        }
    }
    // init boots
    // 实例化自定义启动文件，实例化时注入app参数，存到生命周期的this[BOOTS]数组中
    this.lifecycle.init();
}

// egg-core/lib/lifecycle.js
// 把app.js或agent.js启动自定义函数包装成类的形式
addFunctionAsBootHook(hook) {
    assert(this[INIT] === false, 'do not add hook when lifecycle has been initialized');
    // app.js is exported as a function
    // call this function in configDidLoad
    this[BOOT_HOOKS].push(class Hook {
        constructor(app) {
            this.app = app;
        }
        // 注意：这里是包装到"配置加载完成"阶段中执行
        configDidLoad() {
            hook(this.app);
        }
    });
}
  

// egg-core/lib/lifecycle.js
// 执行 "配置文件即将加载"阶段，执行 configWillLoad 方法
triggerConfigWillLoad() {
    // 执行configWillLoad方法
    for (const boot of this[BOOTS]) {
        if (boot.configWillLoad) {
            boot.configWillLoad();
        }
    }

    // 继续执行下一个阶段，"配置文件加载完成"阶段
    this.triggerConfigDidLoad();
}
```



## 5.2 configDidLoad

上一阶段 `configWillLoad` 执行结束后，会接着调用 `triggerConfigDidLoad` 触发生命周期的第二个阶段（`configDidLoad` 阶段），即配置文件加载完成阶段，在这个阶段中

* 先遍历 `BOOTS` 数组，依次判断每个对象是否存在 `configDidLoad` 方法，存在则调用该方法（它也是一个同步函数）
* 接着判断该对象是否定义了 `beforeClose` 应用关闭的方法，存在就把该方法存放到关闭阶段的数组里
* 最后接着触发生命周期的下一个阶段



**源码如下**

```js
// egg-core/lib/lifecycle.js 
// "配置文件加载完成"阶段，执行configDidLoad方法
triggerConfigDidLoad() {
   // 执行 configDidLoad 方法
    for (const boot of this[BOOTS]) {
        if (boot.configDidLoad) {
            boot.configDidLoad();
        }
        // function boot hook register after configDidLoad trigger
        const beforeClose = boot.beforeClose && boot.beforeClose.bind(boot);
        if (beforeClose) {
            this.registerBeforeClose(beforeClose);
        }
    }
    
    // 继续执行下一个阶段，"文件加载完成"阶段
    this.triggerDidLoad();
}
```



## 5.3 didLoad

上一阶段 `configDidLoad` 阶段结束后，会接着调用 `triggerDidLoad` 触发第三个阶段（`didLoad` 阶段），即文件加载完成阶段，在这个阶段中

* 先遍历 `BOOTS` 数组，依次判断每个对象是否存在 `didLoad` 方法，存在则注册一个回调（它是异步函数，这里主要用到了 `ready-callback` 库来控制流程）
* 注册完回调后，会调用 `process.nextTick`，在里面会执行 `didLoad` 方法，执行完毕后，上面注册的回调被标识为已完成，如果所有注册的 `didLoad` 都执行完了，会触发开始生命周期的下一个阶段（` willReady` 阶段）



**注意：**

* 这里使用了 `ready-callback` 库来控制流程，`ready-callback` 里面又用了 `get-ready` 库，这里用的是 `this.loadReady` 对象把每个 `didLoad` 方法注册为一个回调，这个  `this.loadReady` 对象在生命周期 `lifecycle` 实例化时就实例化好了，它是一个 `ready-callback` 实例
* 当同步代码执行完毕，事件循环转到下一阶段，会开始执行 `process.nextTick` 的任务，里面的任务是 `didLoad` 异步方法，执行完成 `didLoad` 方法，再调用 `this.loadReady` 的 `done()` 方法。通过分析 `ready-callback` 的源码可以知道执行 `done()` 时会判断  `this.loadReady`  所有注册的回调是否执行完了，执行完了就会触发执行  `this.loadReady` 上的`ready()`函数，即执行 `this.loadReady.ready()`， `ready()` 里面会接着触发下一个生命周期阶段的执行 `this.triggerWillReady()`



**源码如下**

```js
// egg-core/lib/lifecycle.js
class Lifecycle extends EventEmitter {
  constructor(options) {
    // 里面会实例化 loadReady
    this[INIT_READY]();
  }

[INIT_READY]() {
    // loadReady 实例化
    this.loadReady = new Ready({ timeout: this.readyTimeout });
  
    this[DELEGATE_READY_EVENT](this.loadReady);
  
  
    // loadReady 注册的回调 didLoad 方法都执行完了之后则会触发这里的执行
    this.loadReady.ready(err => {
      debug('didLoad done');
      if (err) {
        this.ready(err);
      } else {
        // 执行下一个生命周期阶段
        this.triggerWillReady();
      }
    });

}

// 执行"文件加载完成"阶段，把didLoad方法注册到this.loadReay中(ready-callback库的Ready实例)
triggerDidLoad() {
    for (const boot of this[BOOTS]) {
        const didLoad = boot.didLoad && boot.didLoad.bind(boot);
        if (didLoad) {
            // 把 didLoad 注册到this.loadReay中
            this[REGISTER_READY_CALLBACK]({
                scope: didLoad,
                ready: this.loadReady,
                timingKeyPrefix: 'Did Load',
                scopeFullName: boot.fullPath + ':didLoad',
            });
        }
    }
}
  
  
// 注册一个回调(作用：可以先执行完一些逻辑，然后调用该服务，此时会执行ready对象中ready方法注册的函数)
[REGISTER_READY_CALLBACK]({ scope, ready, timingKeyPrefix, scopeFullName }) {
    if (!is.function(scope)) {
        throw new Error('boot only support function');
    }

    // get filename from stack if scopeFullName is undefined
    const name = scopeFullName || utils.getCalleeFromStack(true, 4);
    const timingkey = `${timingKeyPrefix} in ` + utils.getResolvedFilename(name, this.app.baseDir);

    this.timing.start(timingkey);

    // 注册一个回调
    const done = ready.readyCallback(name);

    // ensure scope executes after load completed
    process.nextTick(() => {
        utils.callFn(scope).then(() => {
            // 服务完成，此时会执行ready对象ready方法注册的函数，此时会触发"插件启动完毕"阶段willReady
            done();
            this.timing.end(timingkey);
        }, err => {
            done(err);
            this.timing.end(timingkey);
        });
    });
}
```



## 5.4 willReady

上一个阶段执行结束后，会调用 `triggerWillReady()` 触发第四个阶段（`willReady` 阶段），即插件启动完毕阶段，在这个阶段中

* 先遍历 `BOOTS` 数组，依次判断每个对象是否存在 `willReady` 方法（它是异步方法），存在则注册一个回调

* 注册完回调后，会调用 `process.nextTick`，在里面会执行所有的 `willReady` 方法，执行完毕后，上面注册的回调被标识为已完成，如果所有注册的 `willReady` 都执行完了，会触发开始生命周期的下一个阶段（` didReady` 阶段）

  

**注意：**这里的实现跟上一个阶段的 `triggerDidLoad` 阶段类似，用到了  `ready-callback`  和 `get-ready` 库来控制流程

* 这个阶段用了 `this.bootReady` 对象把每个 `willReady` 方法注册一个回调，它也是一个 `ready-callback` 库的对象，这个 `this.bootReady` 对象也是在生命周期 `lifecycle` 实例化时就实例化好了

* 当同步代码执行完毕，事件循环转到下一阶段，开始执行 `process.nextTick` 的任务，里面的任务就是 `w illReady` 异步方法，先执行完成 `willReady` 方法，再调用  `this.bootReady` 的 `done()`。通过分析 `ready-callback` 的源码可以知道执行 `done()` 时会判断  `this.bootReady`  所有注册的回调是否执行完了，执行完了就会触发执行  `this.bootReady`上的 `ready()`函数，即执行 `this.bootReady.ready()`，  `ready()` 里面会执行 `this.ready(true)`，这个 `this` 是生命周期对象 `lifecycle`，生命周期对象被 `get-ready` 库包装了一下，所以调用 `this.ready(true)` 会执行生命周期对象上的 `ready()` 方法上的回调，里面会触发下一个生命周期的执行 `this.triggerDidReady()`



**源码如下**

```js
// egg-core/lib/lifecycle.js
// 触发"插件启动完毕"阶段willReady
triggerWillReady() {
    this.bootReady.start();
    for (const boot of this[BOOTS]) {
        const willReady = boot.willReady && boot.willReady.bind(boot);
        if (willReady) {
            // 注册回调
            this[REGISTER_READY_CALLBACK]({
                scope: willReady,
                ready: this.bootReady,
                timingKeyPrefix: 'Will Ready',
                scopeFullName: boot.fullPath + ':willReady',
            });
        }
    }
}

// egg-core/lib/lifecycle.js
class Lifecycle extends EventEmitter {
  constructor(options) {
    // lifecycle使用了get-ready库包装了一下
    getReady.mixin(this);

    // 实例化 bootReady
    this[INIT_READY]();
    
    // 注册一个回调，当调用 this.ready(true) 时会执行此回调
    this.ready(err => {
      // 开启didReady阶段
      this.triggerDidReady(err);
      this.timing.end('Application Start');
    });
  }

[INIT_READY]() {
    // bootReady 实例化
    this.bootReady = new Ready({ timeout: this.readyTimeout, lazyStart: true });
    this[DELEGATE_READY_EVENT](this.bootReady);
    // 调用 this.ready，会触发构造方式通过 this.ready() 注册的函数
    this.bootReady.ready(err => {
      // 触发 lifecycle 上 ready() 的回调 
      this.ready(err || true);
    });
}
```



## 5.5 didReady

上一个阶段执行结束后，会调用 `triggerDidReady()` 触发第五个阶段（`didReady` 阶段），即 `worker` 准备就绪阶段，在这个阶段中

* 先遍历 `BOOTS` 数组，依次判断每个对象是否存在 `didReady` 方法，有则执行（这个方法也是异步）
* 这个阶段执行后也不会继续触发下一个阶段了，下一个阶段是在应用启动完成时被触发的



**源码如下**

```js
// 触发"worker 准备就绪"阶段didReady
triggerDidReady(err) {
    (async () => {
        for (const boot of this[BOOTS]) {
            if (boot.didReady) {
                try {
                  // 执行 didReady 函数
                    await boot.didReady(err);
                } catch (e) {
                    this.emit('error', e);
                }
            }
        }
    })();
}
```



## 5.6 serverDidReady

下一个阶段是 `serverDidReady ` 阶段，即应用启动完成阶段。该阶段是在应用启动后，通过接收 `egg-ready` 消息后触发的，当 `egg` 所有 `App` 进程启动完毕后，会发 `egg-ready` 事件通知各个进程，此时每个 `App` 进程监听到 `egg-ready` 事件后就会开始触发生命周期函数的“应用启动完成”阶段，在这个阶段中

* 先遍历 `BOOTS` 数组，依次判断每个对象是否存在 `serverDidReady` 方法（异步方法），存在则执行
* 这个阶段后也不会继续出发下一个阶段的执行，因为下一个是关闭阶段，只有关闭时才会调用



**源码如下**

```js
// egg/lib/egg.js
// trigger `serverDidReady` hook when all the app workers
// and agent worker are ready
this.messenger.once('egg-ready', () => {
    // 触发生命周期的 didReady阶段
    this.lifecycle.triggerServerDidReady();
});


// 触发“应用启动完成”-serverDidReady
triggerServerDidReady() {
    (async () => {
        for (const boot of this[BOOTS]) {
            try {
                // 执行serverDidReady
                await utils.callFn(boot.serverDidReady, null, boot);
            } catch (e) {
                this.emit('error', e);
            }
        }
    })();
}
```



## 5.7 beforeClose

最后的阶段是 `beforeClose` 阶段，即应用即将关闭阶段。`beforeClose` 是在 `app/agent` 实例的 `close` 方法被调用时才触发调用的

* 前面讲了它是在"配置文件加载完成阶段（ `configDidLoad` 阶段）"注册的，存放在 `CLOSE_SET` 数组中

* 接着按注册的顺序，逆序执行每个 `beforeClose`（也是异步方法）



**注意：**

* 这个方法一般用于资源的释放操作，例如 `egg` 用来关闭 `logger`、删除监听方法等
* 开发者不应该直接使用 `app.beforeClose`， 而是定义类的形式，实现 `beforeClose` 方法
* 这个方法不建议在生产环境使用，可能遇到未执行完就结束进程的问题



**源码如下**

```js
// egg-core/lib/egg.js  
async close() {
    if (this[CLOSE_PROMISE]) return this[CLOSE_PROMISE];
    // 
    this[CLOSE_PROMISE] = this.lifecycle.close();
    return this[CLOSE_PROMISE];
 }

// egg-core/lib/lifecycle.js
 async close() {
    // close in reverse order: first created, last closed
    const closeFns = Array.from(this[CLOSE_SET]);
    // 倒序
    for (const fn of closeFns.reverse()) {
      await utils.callFn(fn);
      this[CLOSE_SET].delete(fn);
    }
    // Be called after other close callbacks
    this.app.emit('close');
    this.removeAllListeners();
    this.app.removeAllListeners();
    this[IS_CLOSED] = true;
  }
```



##  5.8 其他：beforeStart

`egg` 的应用类还提供了一个 `beforeStart` 方法，新版已经不建议使用该方法。在使用新版本后，对于应用开发而言，应该用 `willReady` 方法代替；对于插件开发，应该用 `didLoad` 代替，例如

```js
// app.js 或 agent.js 文件：
class AppBootHook {
  constructor(app) {
    this.app = app;
  }

  async didLoad() {
    // 请将你的插件项目中 app.beforeStart 中的代码置于此处
  }
  
  async willReady() {
    // 请将你的应用项目中 app.beforeStart 中的代码置于此处
  }
}

module.exports = AppBootHook;
```

以前旧版本中，是这样使用 `beforeStart` 的，在 `app.js` 中通过 `module.exports` 中传入的 `app` 参数进行此函数的操作，例如

```js
module.exports = (app) => {
  app.beforeStart(async () => {
    // 此处是你原来的逻辑代码
  });
};
```

 `beforeStart` 方法主要是用于注册一些异步方法，实际 `beforeStart` 的实现也是通过往 `this.loadReady` 注册一个回调，跟 `didLoad` 阶段的实现一样的。通常用于执行一些异步任务，例如检查连接状态等。例如，`egg-mysql` 使用 `beforeStart` 来检查 `MySQL` 的连接状态。所有 `beforeStart` 任务结束后，应用将进入 `ready` 状态。不建议执行耗时长的方法，可能导致应用启动超时。



**beforeStart 源码如下**

```js
// egg-core/lib/egg.js
class EggCore extends KoaApplication {
  beforeStart(scope, name) {
    // 调用了生命周期的registerBeforeStart函数
    this.lifecycle.registerBeforeStart(scope, name || '');
  }
}

// egg-lib/lifecycle.js
 registerBeforeStart(scope, name) {
    this[REGISTER_READY_CALLBACK]({
      scope,
      ready: this.loadReady, // 也是用了 loadReady，注册为loadReady回调
      timingKeyPrefix: 'Before Start',
      scopeFullName: name,
    });
  }
```



# 6 总结

本文主要讲了 `Egg` 的加载器机制，在了解加载器具体实现前，也先讲解了 `Egg` 中框架、插件、加载器基础 `API` 等知识点，在加载器具体的加载过程中，也涉及了 `Egg` 的生命周期



# 7 参考

* [egg文档-框架扩展](https://www.eggjs.org/zh-CN/basics/extend)
* [egg文档-启动自定义](https://www.eggjs.org/zh-CN/basics/app-start)
* [egg文档-加载器](https://www.eggjs.org/zh-CN/advanced/loader)
* [egg文档-插件开发](https://www.eggjs.org/zh-CN/advanced/plugin)
* [egg-文档-框架开发](https://www.eggjs.org/zh-CN/advanced/framework)

