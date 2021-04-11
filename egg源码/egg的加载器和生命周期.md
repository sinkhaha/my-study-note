# 一、app和agent类

> egg包提供了app类和agent类的相关处理

> egg/index.js

```js
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



## 1、继承关系

* Agent 和 Application 都继承自 `EggApplication`

* EggApplication继承自`EggCore`
* EggCore继承自`KoaApplication(即koa)`



![egg类图](/Users/shuxinlin/Downloads/egg类图.png)

## 2、EggApplicaion类

> egg/lib/egg.js

1. `loadConfig()`加载配置
2. 实例app和agent之间`messenger通信`的对象
3. 监听`egg-ready`，触发生命周期`应用启动完成`函数`triggerServerDidReady()`
4. 注册一些方法

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

        // 注册方法，把加载完的配置和各阶段的执行时间写入文件中，方便排查问题
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



## 3、EggCore类

> egg-core/lib/egg.js

1. 实例化`生命周期对象`
2. 实例化`加载器`

```js

class EggCore extends KoaApplication {
    constructor(ctx) {
        super();

        // 平时继承的Controller和Service 类其实是BaseContextClass的一个别称
        // 所以可以在Controller和Service中调用this.ctx对象
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

> egg-core/lib/utils/base_context_class.js的BaseContextClass类

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

> egg-core/lib/loader/egg_loader.js

是egg的加载器，是AgentWorkerLoader和AppWorkerLoader加载器的父类，定义了一些约定

```js
class EggLoader {
    // 获取加载单元
    getLoadUnits() {
        // 按 插件->框架->应用 依次加载
    }
}

// 即egg中加载器各方法处理的主要实现，各实现在egg-core/lib/loader/mixin/目录下
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

## 4、Agent类

> egg/lib/agent.js

```js

class Agent extends EggApplication {
    constructor(options = {}) {
        super(options);
        // 调用agent加载器的load方法，也是egg加载器中核心的加载逻辑
        this.loader.load(); 
    }
  
    // agent加载器
    get [EGG_LOADER]() {
        return AgentWorkerLoader;
    }
}
```



### Agent加载器

> egg/lib/loader/agent_worker_loader.js

```js
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
        this.loadAgentExtend(); // 框架扩展，加载app/extend/agent.js到app上
        this.loadContextExtend(); // 框架扩展，加载app/extend/context.js到app上

        this.loadCustomAgent();  // 加载启动自定义，会开始生命周期
    }
}
```



## 5、Application类

> egg/lib/application.js

```js
// app加载
class Application extends EggApplication { 
    constructor(options = {}) {
        super(options);
        // 调用app加载的load方法
        this.loader.load();
    }
  
    get [EGG_LOADER]() {
        return AppWorkerLoader;
    }
}
```



### Application加载器

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



# 二、egg加载器各方法实现

![](https://gitee.com/sinkhaha/picture/raw/master/img/egg/egg%E5%8A%A0%E8%BD%BD%E5%99%A8.png)



> 源码位于egg-core/lib/loader/mixin/目录下

## 1、插件plugin加载

### 加载流程

1. 读取所有插件的配置

   （1）读取`当前应用目录`下的插件配置

   （2）读取`所有framework框架目录`的插件配，如egg框架的默认插件

   （3）读取`自定义插件`，由于`process.env.EGG_PLUGINS` 和 `options.plugins`指定

2. 合并所有插件的配置，按`框架插件-> 应用插件 -> 自定义插件`的顺序合并

3. 根据`插件配置`读取插件所在的库，然后读取库中`package.json`配置信息，扩展插件信息，过滤掉不属于当前环境的插件

4. 排序所有插件（过程中会解决依赖等问题）

5. 把插件挂载在加载器loader的plugins属性上


**注意：**对于同名的插件，在后面读取的插件的各属性会赋值给前面的插件的同名属性，所以相同名的属性会被覆盖



### 源码分析

> egg-core/lib/loader/mixin/plugin.js

```js
loadPlugin() {
    // 1.1 读取当前项目的插件配置
    // -> readPluginConfigs(configPaths)会去依次加载传入路径下
    //    ->plugin.default.js  
    //    -> plugin.${serverScope}.js 
    //    -> plugin.${serverEnv}.js 
    //    -> plugin.${serverScope}_${serverEnv}.js文件下的插件
    // ->对于同名的插件，在后面读取的插件的各属性会赋值给前面的插件的同名属性，所以相同名的属性会被覆盖
    const appPlugins = this.readPluginConfigs(path.join(this.options.baseDir, 'config/plugin.default'));

    // 1.2 读取所有框架framework目录的默认插件配置，如读取egg框架自带的默认插件
    const eggPluginConfigPaths = this.eggPaths.map(eggPath => path.join(eggPath, 'config/plugin.default'));
    const eggPlugins = this.readPluginConfigs(eggPluginConfigPaths);

    // 1.3 读取自定义插件，包括process.env.EGG_PLUGINS 和 options.plugins指定的插件，options.plugins会覆盖process.env.EGG_PLUGINS同名的插件配置
    let customPlugins;
    if (process.env.EGG_PLUGINS) {
        try {
            customPlugins = JSON.parse(process.env.EGG_PLUGINS);
        } catch (e) {
            debug('parse EGG_PLUGINS failed, %s', e);
        }
    }
    // loader plugins from options.plugins
    if (this.options.plugins) {
        customPlugins = Object.assign({}, customPlugins, this.options.plugins);
    }
    if (customPlugins) {
        for (const name in customPlugins) {
            // 补全各个插件对象的配置
            this.normalizePluginConfig(customPlugins, name);
        }
    }
    ......
    
    // 2. 合并读取到的插件配置
    // 按框架 插件 -> 应用插件 -> 自定义插件 的顺序合并读取到的插件
    // 对于重复定义的同名的插件，在后面读取的插件的各属性会赋值给前面的插件的同名属性，所以相同名的属性会被覆盖
    this._extendPlugins(this.allPlugins, eggPlugins);
    this._extendPlugins(this.allPlugins, appPlugins);
    this._extendPlugins(this.allPlugins, customPlugins);

    // 3. 根据插件配置读取插件所在的库，读取package.json配置信息，扩展插件信息，过滤掉不属于当前环境的插件
    for (const name in this.allPlugins) {
        const plugin = this.allPlugins[name];

        // getPluginPath函数找到插件所在的库
        // （1）先找配置中的path，
        // （2）否则根据package或name名找到所在的库。
        //  其查找规则如下  
        //  -> {APP_PATH}/node_modules  先找项目下的node_modules  
        //    -> {EGG_PATH}/node_modules 再找框架下的node_modules，egg的框架是可以无限继承的，所以框架优先从外往里查找
        //      -> $CWD/node_modules 当前路径下的node_modules
        plugin.path = this.getPluginPath(plugin, this.options.baseDir);

        // 读取插件库中的package.json里eggPlugin等配置信息，并把配置合并到插件中
        // read plugin information from ${plugin.path}/package.json
        this.mergePluginConfig(plugin);

        // 屏蔽不属于当前环境下的插件   
        if (env && plugin.env.length && !plugin.env.includes(env)) {
            plugin.enable = false;
            continue;
        }

        // 最后所有的插件
        plugins[name] = plugin;
        // 启用的插件名字
        if (plugin.enable) {
            enabledPluginNames.push(name);
        }
    }

    // 4. 排序所有插件 
    // 根据依赖等信息，递归调用sequence函数获得插件的加载顺序(会处理循环依赖，依赖缺失等问题）
    this.orderPlugins = this.getOrderPlugins(plugins, enabledPluginNames);
     
    ......
    // 5. 挂载插件
    this.plugins = enablePlugins;
}

```



### 插件的作用

1. 扩展内置对象(app/extend/)

   > 如app/extend/application.js- 扩展 Application 类

2. 插入自定义中间件 

   > ```
   > 如：
   > app.config.coreMiddleware.splice(index, 0, '中间经名');
   > ```

3. 在应用启动时做一些初始化工作

   > app.js或agent.js中做一些工作，如调用beforeStart方法等操作

   

## 2、配置config加载

### 加载流程

1. 加载当前项目的配置

   > 先加载config.default文件，后加载config.${this.serverEnv}文件，同名属性会覆盖

2. 按`读取到的所有文件的名字 和 `插件 -> 框架 -> 应用`的顺序，加载配置

   > plugin config.default
   >     framework config.default
   >         app config.default
   >             plugin config.{env}
   >               framework config.{env}
   >                 app config.{env}

3. 加载环境变量的配置`process.env.EGG_APP_CONFIG`

4. 加载中间件的配置到config对象的`coreMiddleware`属性和`appMiddleware`属性中

   > `框架和插件`不支持在 `config.default.js` 中匹配 `middleware`可以在app.js中
   >
   > `app.config.coreMiddleware.unshift('中间件名')`的方式使用中间件

5. 把配置挂载到config属性

   

**注意：**配置有同名属性会进行`覆盖`，数组也是覆盖而不是合并，具体由`extend2`包实现  



### 源码分析

> egg-core/lib/loader/mixin/config.js

```js
loadConfig() {
    // 1. 加载当前项目的配置
    // ->先加载config.default，后加载config.${this.serverEnv}的，出现同名属性，后面的会直接覆盖掉前面的(具体实现为extend2包)
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



## 3、扩展文件extend加载

> egg-core/lib/loader/mixin/extend.js

加载 `插件/框架/项目` 下`app/extend/目录下`的文件到对应的对象上

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



## 4、custom启动自定义(生命周期)

### 生命周期函数

![](https://gitee.com/sinkhaha/picture/raw/master/img/egg/%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E5%87%BD%E6%95%B0.png)



**以启动app为例，整个启动流程大致为：**

1. egg包中的实例化Application后调用`this.loader.load()`加载，load方法中调用了loadCustomApp方法

2. loadCustomApp方法加载`所有加载单元`根目录下的app.js，此时触发`配置文件即将加载`阶段-`configWillLoad`，开始生命周期

   > 遍历所有加载单元自定义启动文件，依次执行configWillLoad方法

3. 触发`文件加载完成`阶段-`configDidLoad`

   >  遍历所有加载单元自定义启动文件，依次执行configDidLoad方法
   >
   >  如果有beforeClose方法，则注册beforeClose方法

4. 触发`文件加载完成`阶段-`didLoad`

   >遍历所有加载单元自定义启动文件，把`didLoad注册`到`this.loadReady方法`中，然后在`process.nextTick`中执行，执行完则开始生命周期下一阶段。
   >
   >(此处和前面的生命周期执行的区别)

5. 触发`插件启动完毕`阶段-`willReady`

   > 遍历所有加载单元自定义启动文件，把`willReady注册`到`this.bootReady`方法中，然后执行完所有willReady，开始生命周期的下一阶段
   >
   > 
   >
   > 说明：在实例化lifecycle时，构造函数中调用`[INIT_READY](){}`就已经注册triggerWillReady方法了，所以在didLoad执行后，就会执行该方法。(注：也可以调用beforeStart注册一些方法在该阶段执行)


6. 触发`worker 准备就绪`阶段-`didReady`

   >遍历自定义启动文件，执行didReady方法
   >
   >
   >
   >说明：在`willReady执行完`，即`this.bootReady`启动回调注册的服务执行完了，就执行`this.bootReady.ready()`方法，里面会调用`this.ready(err || true)`，
   >
   >所以最后会触发到ready注册的回调，即执行`this.triggerDidReady(err))`开始didRead生命周期

7. 触发`应用启动完成`阶段-`serverDidReady`

   > 在egg-cluster库中触发消息，发送egg-ready事件，`egg/lib/egg.js`接收到egg-ready消息开始serverDidReady生命周期
   >
   >  this.messenger.once('egg-ready', () => {
   >
   > ​      this.lifecycle.triggerServerDidReady();
   >
   >  });

8. 触发`应用即将关闭`阶段-`beforeClose`

   > 是在configDidLoad阶段才注册的，在 app/agent 实例的 `close` 方法被调用后, 按注册的逆序执行。
   >
   > 一般用于资源的释放操作, 例如 egg用来关闭 logger、删除监听方法等。开发者不应该直接使用 `app.beforeClose`, 而是定义类的形式, 实现 `beforeClose` 方法
   >
   > 
>这个方法不建议在生产环境使用, 可能遇到未执行完就结束进程的问题。
>
>在框架的进程关闭处理中是有超时时间的，如果 worker 进程在接收到进程退出信号之后，没有在所规定的时间内退出，将会被强制关闭。


9. 最后才执行app_worker.js文件`ready注册的方法`，即`app.ready(startServer)`



**beforeStart方法**

把beforeStart方法注册到`this.loadReady`中，和`willReady`阶段一样。此方法在加载过程中被调用，所有的方法并行执行，一般用来执行一些异步方法，例如检查连接状态等, 比如egg-mysql 就用 `beforeStart` 来检查与 mysql 的连接状态。

所有的 `beforeStart` 任务结束后，状态将会进入 `ready` 。不建议执行一些`耗时较长`的方法, 可能会导致`应用启动超时`。插件开发者应使用 `didLoad` 



### 源码分析

> egg-core/lib/loader/mixin/custom.js

```js
// 加载`所有加载单元`根目录下的app.js定义的周期函数，并触发'配置文件即将加载configWillLoad'阶段
loadCustomApp() {
    this[LOAD_BOOT_HOOK]('app');
    // 触发'配置文件即将加载configWillLoad'阶段，开始生命周期
    this.lifecycle.triggerConfigWillLoad();
},

// 加载`所有加载单元`根目录下的agent.js定义的周期函数，并触发'配置文件即将加载configWillLoad'阶段
loadCustomAgent() {
    this[LOAD_BOOT_HOOK]('agent');
    this.lifecycle.triggerConfigWillLoad();
},

// 加载`所有加载单元`下的app.js或agent.js，然后实例化
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
        // 包装到"配置加载完成"阶段中执行
        configDidLoad() {
            hook(this.app);
        }
    });
}
```

> egg-core/lib/lifecycle.js

```js
// 1. 执行 "配置文件即将加载"阶段，执行 configWillLoad 方法
triggerConfigWillLoad() {
    // 执行configWillLoad方法
    for (const boot of this[BOOTS]) {
        if (boot.configWillLoad) {
            boot.configWillLoad();
        }
    }

    // 执行"配置文件加载完成"阶段
    this.triggerConfigDidLoad();
}

// 2.执行"配置文件加载完成"阶段，执行configDidLoad方法
triggerConfigDidLoad() {
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
    // 执行"文件加载完成"阶段
    this.triggerDidLoad();
}

// 3.执行"文件加载完成"阶段，把didLoad方法注册到this.loadReay中(ready-callback库的Ready实例)
triggerDidLoad() {
    for (const boot of this[BOOTS]) {
        const didLoad = boot.didLoad && boot.didLoad.bind(boot);
        if (didLoad) {
            // 注册到this.loadReay中
            this[REGISTER_READY_CALLBACK]({
                scope: didLoad,
                ready: this.loadReady,
                timingKeyPrefix: 'Did Load',
                scopeFullName: boot.fullPath + ':didLoad',
            });
        }
    }
}

// 注册一个服务(作用：可以先执行完一些逻辑，然后调用该服务，此时会执行ready对象中ready方法注册的函数)
[REGISTER_READY_CALLBACK]({ scope, ready, timingKeyPrefix, scopeFullName }) {
    if (!is.function(scope)) {
        throw new Error('boot only support function');
    }

    // get filename from stack if scopeFullName is undefined
    const name = scopeFullName || utils.getCalleeFromStack(true, 4);
    const timingkey = `${timingKeyPrefix} in ` + utils.getResolvedFilename(name, this.app.baseDir);

    this.timing.start(timingkey);

    // 注册一个服务
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


// 触发"插件启动完毕"阶段willReady的入口
[INIT_READY]() {
    this.loadReady = new Ready({ timeout: this.readyTimeout });
    this[DELEGATE_READY_EVENT](this.loadReady);
    // 当this.loadReady注册的服务done完成时，则会执行该回调开始生命周期的下一阶段
    this.loadReady.ready(err => {
        debug('didLoad done');
        if (err) {
            this.ready(err);
        } else {
            // 触发"插件启动完毕"阶段willReady
            this.triggerWillReady();
        }
    });

    this.bootReady = new Ready({ timeout: this.readyTimeout, lazyStart: true });
    this[DELEGATE_READY_EVENT](this.bootReady);
    this.bootReady.ready(err => {
        this.ready(err || true);
    });
}

ready(flagOrFunction) {
    return this.lifecycle.ready(flagOrFunction);
}

// 4.触发"插件启动完毕"阶段willReady
triggerWillReady() {
    this.bootReady.start();
    for (const boot of this[BOOTS]) {
        const willReady = boot.willReady && boot.willReady.bind(boot);
        if (willReady) {
            this[REGISTER_READY_CALLBACK]({
                scope: willReady,
                ready: this.bootReady,
                timingKeyPrefix: 'Will Ready',
                scopeFullName: boot.fullPath + ':willReady',
            });
        }
    }
}

this.ready(err => {
    this.triggerDidReady(err);
    this.timing.end('Application Start');
});

// 5.触发"worker 准备就绪"阶段didReady
triggerDidReady(err) {
    (async () => {
        for (const boot of this[BOOTS]) {
            if (boot.didReady) {
                try {
                    await boot.didReady(err);
                } catch (e) {
                    this.emit('error', e);
                }
            }
        }
    })();
}

// 6.触发“应用启动完成”-serverDidReady
triggerServerDidReady() {
    (async () => {
        for (const boot of this[BOOTS]) {
            try {
                await utils.callFn(boot.serverDidReady, null, boot);
            } catch (e) {
                this.emit('error', e);
            }
        }
    })();
}


```

> egg-core/lib/egg.js

```js
async close() {
    if (this[CLOSE_PROMISE]) return this[CLOSE_PROMISE];
    // 7.应用即将关闭阶段-beforeClose
    this[CLOSE_PROMISE] = this.lifecycle.close();
    return this[CLOSE_PROMISE];
}
```



## 5、中间件middleware加载

### 加载流程

1. 获取`所有加载单元`的`app/middleware目录`(插件->框架 ->应用)

2. 将`middlewares目录`中的中间件读取到`app.middlewares`属性中

3. 然后遍历app.middlewares把中间件定义到`app.middleware`属性上

4. 根据`配置文件`中`中间件的配置`加载中间件

   > app.config.coreMiddleware和app.config.appMiddleware属性的配置；
   >
   > 在配置加载阶段就已经加载了配置中的coreMiddleware和middleware

5. use使用中间件

注意：`框架`和`插件`加载的中间件会在`应用层`配置的中间件之前，框架默认中间件`不能`被应用层中间件覆盖，如果应用层有自定义`同名中间件`，在启动时会`报错`

### 源码分析

> egg-core/lib/loader/mixin/middleware.js

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

## 6、读取service

1. 依然还是按加载单元的顺序`插件->框架->应用`加载`app/service`目录下的所有文件
2. 把所有service挂载到`context对象`上，即`ctx.service.xxx`



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

## 7、读取controller

**插件和框架没有controller**

所以只需要读取当前项目的`app/controller`文件下的文件，然后注册到`app. controller`对象上



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

## 8、读取路由router

**插件和框架没有路由**

所以只需要加载`当前项目`的`app/router.js`文件即可



> egg-core/lib/loader/mixin/router.js

```js
loadRouter() {
    // 加载 router.js  在app下的router文件中定义，所以直接读取代码即可
    this.loadFile(path.join(this.options.baseDir, 'app/router'));
}
```

## 9、自定义加载器custom_loader

> egg-core/lib/loader/mixin/custom_loader.js

可以自定义加载方法`loadToContext和loadToApp方法`，具体用法可参考[egg官方文档](https://eggjs.org/zh-cn/advanced/loader.html)



## 参考

* https://www.yuque.com/jianxu/study/suablh

