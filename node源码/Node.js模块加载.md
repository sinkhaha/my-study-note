# 一、Node.js模块的分类

## 1、用户模块

### 用户源码模块

项目中导入的js源码模块、第三方包、json模块等



### c++扩展(.node)

1. 用户自己写的c++扩展模块，一个编译好的c++扩展模块后缀名是*.node*，如`test.node`
2. 编译好的c++扩展模块是一个动态链接库，如

> windows下的*.dll
>
> linux下的*.so
>
> mac下的*.dylib

3. Node.js引入一个c++扩展模块其实就是在Node.js运行时引入一个动态链接库的过程



## 2、Node.js内置(原生)模块

### Node.js内置c++模块(built-in模块)

1. 内置c++模块存在于Node.js源码中

   > 如src/tcp_wrap.cc文件即tcp_wrap模块

2. 它被编译进Node.js的可执行二进制文件中，而不像用户的c++扩展模块一样以动态链接库的形式存在

3. 一般不被用户直接调用，而是提供给Node.js内置js模块调用



### Node.js原生js模块(native模块)

1. 内置js模块等同于官方文档中的模块，如fs、path等等

2. 在Node.js源码，内置js模块大多在lib目录下以同名js文件的形式被实现（这些lib下的js文件最后被编译进Node.js里），并且很多Node.js内置js模块是对c++内置模块的一个封装



# 二、CommonJS模块

## CommonJS规范对一个模块的约定

1. require函数：用于加载模块，该函数接收一个模块标识符，其结果为目标模块module.exports导出的API

2. exports对象：在 exports 上挂载的属性会被导出

3. module对象：module代表当前模块的信息，module对象拥有一个id属性。Node.js中module对象也有一个exports对象，它跟上面第2点的exports对象指向相同

   >  注意：在Node.js中模块真正暴露出去的是module中的exports对象



## CommonJS模块的特点

* 每个文件是一个模块，文件内的对象都是私有的，即所有代码都运行在模块作用域，不会污染全局作用域

- 模块可以被多次加载，但在第一次加载后就被缓存了，再次加载会直接从缓存读取

- 模块加载是同步的，按照其在代码中的顺序加载的，只有加载完成，才能执行后面的操作

  

## Node.js模块的实现

1. Module模块类定义了require函数用于加载模块
2. 每个文件都是一个独立的模块，模块被加载时，都会初始化为 module 对象的实例。实际Node.js中每一个模块对外暴露的是自己的exports属性(即module.exports)。module.exports跟CommonJS规范的exports对象指向是一致的，直接在exports对象上挂载对象也能导出该对象



### 例1：用exports导出

```js
// module-test1.js文件
// 1、直接在exports对象上挂载属性或函数，用于导出属性或函数
exports.name = 'zhangsan';

// console.log(module.exports.name); // zhangsan

exports.add = (a, b) => a + b;

// 为exports赋予新的值，此时不再跟module.exports指向同一个地址，所以在exports新挂载的属性或函数不会被导出
exports = {};
exports.age = 18; // 此时age不会被导出

// test1.js文件
// 导入module-test1.js模块
const { add, name, age } = require('./module-test1.js');

console.log(add(1, 2)); // 3

console.log(name); // zhangsan

console.log(age); // undefined
```

### 例2：用module.exports导出

```js
// module-test2.js文件
const name = 'zhangsan';
const add = (a, b) => a + b;

// 使用module.exports指向新的对象，导出属性或函数
module.exports = {
  add,
  name
}

// test2.js文件
// 导入module-test2.js模块
const { add, name } = require('./module-test2.js');

console.log(add(1, 2)); // 3

console.log(name); // zhangsan
```



# 三、Node.js模块加载源码分析

> 源码基于Node.js v16.18.0

## Module类和require导入

1. Node.js中用Module类存储当前模块的信息
2. require是用来导入模块的语句，是Module类实例的一个方法，内部又调用了Module._load方法

**Module类**

> 源码如下

```js
// lib/internal/modules/cjs/loader.js

// Module类，表示模块的信息
function Module(id = '', parent) {
  // 模块id，一般为模块的绝对路径
  this.id = id; 
  this.path = path.dirname(id);
  // exports属性
  setOwnProperty(this, 'exports', {});
  
  // parent为当前模块的调用者
  // 设置父模块的缓存
  moduleParentCache.set(this, parent);
  // 更新父模块的子模块
  updateChildren(parent, this, false);
  
  this.filename = null;
  
  // 模块是否加载完成
  this.loaded = false;
  
  // 当前模块所引用的模块
  this.children = [];
}
```

**require方法**

> 源码如下

```js
// lib/internal/modules/cjs/loader.js

// 用户使用Module的require导入模块，实际内部是调用了Module._load加载模块
// Loads a module at the given file path. Returns that module's
// `exports` property.
Module.prototype.require = function(id) {
  validateString(id, 'id');
  if (id === '') {
    throw new ERR_INVALID_ARG_VALUE('id', id,
                                    'must be a non-empty string');
  }
  requireDepth++;
  try {
    // 加载模块，isMain表示是否是入口模块，此时为false，返回module.exports对象
    return Module._load(id, this, /* isMain */ false);
  } finally {
    requireDepth--;
  }
};
```



## Module._load实现

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/1%E3%80%81Modul_load%E4%B8%BB%E6%B5%81%E7%A8%8B.png)



1. 存在parent(使用require导入的文件对应的模块对象)，且存在该路径的缓存，则返回module.exports

2. 模块标识是 `node: `开头，说明是内置模块，加载后返回module.exports

3. 解析出模块的全部路径Module._resolveFilename

4. 加载模块

   * 模块已存在缓存Module._cache[filename]，返回module.exports
   * 是核心内置模块，则加载内置模块loadNativeModule，然后返回module.exports
   * 实例化模块Module实例，对模块进行缓存，使用Module.load加载模块，返回module.exports

   

> 源码如下

```js
// lib/internal/modules/cjs/loader.js
// 加载模块的核心逻辑

// Check the cache for the requested file.
// 1. If a module already exists in the cache: return its exports object.
// 2. If the module is native: call
//    `NativeModule.prototype.compileForPublicLoader()` and return the exports.
// 3. Otherwise, create a new module for the file and save it to the cache.
//    Then have it load  the file contents before returning its exports
//    object.
Module._load = function(request, parent, isMain) {
  let relResolveCacheIdentifier;
  if (parent) {
    // Fast path for (lazy loaded) modules in the same directory. The indirect
    // caching is required to allow cache invalidation without changing the old
    // cache key names.
    relResolveCacheIdentifier = `${parent.path}\x00${request}`;
    //  是否有路径缓存
    const filename = relativeResolveCache[relResolveCacheIdentifier];
    if (filename !== undefined) {
      // 是否有缓存
      const cachedModule = Module._cache[filename];
      if (cachedModule !== undefined) {
        updateChildren(parent, cachedModule, true);
        if (!cachedModule.loaded)
          return getExportsForCircularRequire(cachedModule);
        return cachedModule.exports;
      }
      delete relativeResolveCache[relResolveCacheIdentifier];
    }
  }

  if (StringPrototypeStartsWith(request, 'node:')) {
    // Slice 'node:' prefix
    const id = StringPrototypeSlice(request, 5);
    
    // node:开头的是内置模块，加载内置模块
    const module = loadNativeModule(id, request);
    if (!module?.canBeRequiredByUsers) {
      throw new ERR_UNKNOWN_BUILTIN_MODULE(request);
    }

    return module.exports;
  }

  // 解析出模块的路径
  const filename = Module._resolveFilename(request, parent, isMain);

  // 查询是否有缓存，有则直接返回
  const cachedModule = Module._cache[filename];
  if (cachedModule !== undefined) {
    updateChildren(parent, cachedModule, true);
    if (!cachedModule.loaded) {
      const parseCachedModule = cjsParseCache.get(cachedModule);
      if (!parseCachedModule || parseCachedModule.loaded)
        return getExportsForCircularRequire(cachedModule);
      parseCachedModule.loaded = true;
    } else {
      return cachedModule.exports; // 已经加载过，直接返回
    }
  }

  // 加载核心内置模块
  const mod = loadNativeModule(filename, request);
  if (mod?.canBeRequiredByUsers &&
      NativeModule.canBeRequiredWithoutScheme(filename)) {
    return mod.exports;
  }

  // 实例化一个模块Module对象
  // Don't call updateChildren(), Module constructor already does.
  const module = cachedModule || new Module(filename, parent);

  // 是否是入口模块，是则标记一下
  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  // module模块加入缓存
  Module._cache[filename] = module;
  if (parent !== undefined) {
    relativeResolveCache[relResolveCacheIdentifier] = filename;
  }

  let threw = true;
  try {
    // 加载模块
    module.load(filename);
    threw = false;
  } finally {
    if (threw) {
      // 报错则清除一些缓存
      delete Module._cache[filename];
      if (parent !== undefined) {
        delete relativeResolveCache[relResolveCacheIdentifier];
        const children = parent?.children;
        if (ArrayIsArray(children)) {
          const index = ArrayPrototypeIndexOf(children, module);
          if (index !== -1) {
            ArrayPrototypeSplice(children, index, 1);
          }
        }
      }
    } else if (module.exports &&
               !isProxy(module.exports) &&
               ObjectGetPrototypeOf(module.exports) ===
                 CircularRequirePrototypeWarningProxy) {
      ObjectSetPrototypeOf(module.exports, ObjectPrototype);
    }
  }

  // 返回module.exports对象
  return module.exports;
};

```

### 1、解析出路径

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221112194245.png)



1. 是node:开头的原生模块，直接返回导入标识
2. 获取到所有可能要查找的路径paths(Module._resolveLookupPaths)
3. 在所有要查找的路径中找到文件的绝对路径(Module._findPath)
4. 返回绝对路径



>  源码如下

```js
// lib/internal/modules/cjs/loader.js
// request为模块标识
Module._resolveFilename = function(request, parent, isMain, options) {
  // node:开头的是原生内置模块，加载模块
  if (
    (
      StringPrototypeStartsWith(request, 'node:') &&
      NativeModule.canBeRequiredByUsers(StringPrototypeSlice(request, 5))
    ) || (
      NativeModule.canBeRequiredByUsers(request) &&
      NativeModule.canBeRequiredWithoutScheme(request)
    )
  ) {
    return request;
  }

  let paths;

  // options.paths用于指定查找的路径
  if (typeof options === 'object' && options !== null) {
    if (ArrayIsArray(options.paths)) {
      const isRelative = StringPrototypeStartsWith(request, './') ||
          StringPrototypeStartsWith(request, '../') ||
          ((isWindows && StringPrototypeStartsWith(request, '.\\')) ||
          StringPrototypeStartsWith(request, '..\\'));

      // 是相对路径直接在paths中查找
      if (isRelative) {
        paths = options.paths;
      } else {
        const fakeParent = new Module('', null);

        paths = [];

        for (let i = 0; i < options.paths.length; i++) {
          const path = options.paths[i];
          // 先查找path当前路径下的node_modules，不存在则依次往上级文件夹查找node_modules文件夹
          fakeParent.paths = Module._nodeModulePaths(path);
          
          // 如console.log(_nodeModulePaths('/Users/zhangsan/nodejs/test-demo'));
          // 输出如下
          // [
          //    '/Users/zhangsan/nodejs/test-demo/node_modules',
          //    '/Users/zhangsan/nodejs/node_modules',
          //    '/Users/zhangsan/node_modules',
          //    '/Users/node_modules',
          //    '/node_modules',
          // ]
          
          // 查找所有的可能路径
          const lookupPaths = Module._resolveLookupPaths(request, fakeParent);

          for (let j = 0; j < lookupPaths.length; j++) {
            if (!ArrayPrototypeIncludes(paths, lookupPaths[j]))
              ArrayPrototypePush(paths, lookupPaths[j]);
          }
        }
      }
    } else if (options.paths === undefined) {
      paths = Module._resolveLookupPaths(request, parent);
    } else {
      throw new ERR_INVALID_ARG_VALUE('options.paths', options.paths);
    }
  } else {
    // 查找所有可能的路径
    paths = Module._resolveLookupPaths(request, parent);
  }

  if (request[0] === '#' && (parent?.filename || parent?.id === '<repl>')) {
    const parentPath = parent?.filename ?? process.cwd() + path.sep;
    const pkg = readPackageScope(parentPath) || {};
    if (pkg.data?.imports != null) {
      try {
        return finalizeEsmResolution(
          packageImportsResolve(request, pathToFileURL(parentPath),
                                cjsConditions), parentPath, request,
          pkg.path);
      } catch (e) {
        if (e.code === 'ERR_MODULE_NOT_FOUND')
          throw createEsmNotFoundErr(request);
        throw e;
      }
    }
  }

  // Try module self resolution first
  const parentPath = trySelfParentPath(parent);
  const selfResolved = trySelf(parentPath, request);
  if (selfResolved) {
    const cacheKey = request + '\x00' +
         (paths.length === 1 ? paths[0] : ArrayPrototypeJoin(paths, '\x00'));
    Module._pathCache[cacheKey] = selfResolved;
    return selfResolved;
  }

  // 获取文件的绝对路径，request为模块标识，paths为所有要查找的路径
  // Look up the filename first, since that's the cache key.
  const filename = Module._findPath(request, paths, isMain, false);

  // 返回文件的绝对路径
  if (filename) return filename;

  const requireStack = [];
  for (let cursor = parent;
    cursor;
    cursor = moduleParentCache.get(cursor)) {
    ArrayPrototypePush(requireStack, cursor.filename || cursor.id);
  }

  // 没有找到模块，则抛出异常
  let message = `Cannot find module '${request}'`;
  if (requireStack.length > 0) {
    message = message + '\nRequire stack:\n- ' +
              ArrayPrototypeJoin(requireStack, '\n- ');
  }
  // eslint-disable-next-line no-restricted-syntax
  const err = new Error(message);
  err.code = 'MODULE_NOT_FOUND';
  err.requireStack = requireStack;
  throw err;
};
```

#### （1）获取要查找的所有路径 _resolveLookupPaths

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221112192451.png)



> 源码如下

```js
// lib/internal/modules/cjs/loader.js

Module._resolveLookupPaths = function(request, parent) {
  if (NativeModule.canBeRequiredByUsers(request) &&
      NativeModule.canBeRequiredWithoutScheme(request)) {
    debug('looking for %j in []', request);
    return null;
  }

  // Check for node modules paths.
  // 不是相对路径
  if (StringPrototypeCharAt(request, 0) !== '.' ||
      (request.length > 1 &&
      StringPrototypeCharAt(request, 1) !== '.' &&
      StringPrototypeCharAt(request, 1) !== '/' &&
      (!isWindows || StringPrototypeCharAt(request, 1) !== '\\'))) {

    // modulePaths变量在Module._initPaths函数中赋值
    let paths = modulePaths;
    if (parent?.paths?.length) {
      paths = ArrayPrototypeConcat(parent.paths, paths);
    }

    debug('looking for %j in %j', request, paths);
    return paths.length > 0 ? paths : null;
  }

  // 使用REPL交互时，返回当前路径 “./” 
  // In REPL, parent.filename is null.
  if (!parent || !parent.id || !parent.filename) {
    // Make require('./path/to/foo') work - normally the path is taken
    // from realpath(__filename) but in REPL there is no filename
    const mainPaths = ['.'];

    debug('looking for %j in %j', request, mainPaths);
    return mainPaths;
  }

  debug('RELATIVE: requested: %s from parent.id %s', request, parent.id);

  // 如果是以相对路径的方式导入，返回父级文件夹路径
  const parentDir = [path.dirname(parent.filename)];
  debug('looking for %j', parentDir);
  return parentDir;
};
```

#### （2）获取文件的绝对路径_findPath

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/20221112194405.png)



1. 生产cacheKey，如果存在缓存直接返回
2. 循环遍历paths
   * 最后一个字符不是/，且导入的是文件，文件存在直接返回，如果文件不存在，则在导入标识后面依次加上.js、.json、.node扩展名进行查找，存在直接返回，不存在则判断是否是目录
   * 如果导入的是目录，先判断当前目录下的package.json是否存在main指定入口文件，存在则返回，不存在则依次判断当前目录下是否存在index.js、index.json、index.node文件
3. 设置cacheKey缓存



> 源码如下

```js
// lib/internal/modules/cjs/loader.js

Module._findPath = function(request, paths, isMain) {
  const absoluteRequest = path.isAbsolute(request);
  if (absoluteRequest) { // 是绝对路径，重新设置paths
    paths = [''];
  } else if (!paths || paths.length === 0) {
    return false;
  }

  // 生成cacheKey
  const cacheKey = request + '\x00' + ArrayPrototypeJoin(paths, '\x00');
  // 存在缓存，直接返回
  const entry = Module._pathCache[cacheKey];
  if (entry)
    return entry;

  let exts;
  let trailingSlash = request.length > 0 &&
    StringPrototypeCharCodeAt(request, request.length - 1) ===
    CHAR_FORWARD_SLASH;
  if (!trailingSlash) {
    trailingSlash = RegExpPrototypeExec(trailingSlashRegex, request) !== null;
  }

  // For each path
  for (let i = 0; i < paths.length; i++) {
    // Don't search further if path doesn't exist
    const curPath = paths[i];
    if (curPath && _stat(curPath) < 1) continue;

    if (!absoluteRequest) {
      const exportsResolved = resolveExports(curPath, request);
      if (exportsResolved)
        return exportsResolved;
    }

    const basePath = path.resolve(curPath, request);
    let filename;

    const rc = _stat(basePath);
    if (!trailingSlash) { // 导入标识的最后一个字符不是 /
      if (rc === 0) {  // File. 表示是文件
        if (!isMain) {
          if (preserveSymlinks) {
            filename = path.resolve(basePath);
          } else {
            filename = toRealPath(basePath);
          }
        } else if (preserveSymlinksMain) {
          // For the main module, we use the preserveSymlinksMain flag instead
          // mainly for backward compatibility, as the preserveSymlinks flag
          // historically has not applied to the main module.  Most likely this
          // was intended to keep .bin/ binaries working, as following those
          // symlinks is usually required for the imports in the corresponding
          // files to resolve; that said, in some use cases following symlinks
          // causes bigger problems which is why the preserveSymlinksMain option
          // is needed.
          filename = path.resolve(basePath);
        } else {
          filename = toRealPath(basePath);
        }
      }

      if (!filename) { // 没有文件名则取默认的扩展名 .js 和 .json 和 .node
        // Try it with each of the extensions
        if (exts === undefined)
          exts = ObjectKeys(Module._extensions);
        filename = tryExtensions(basePath, exts, isMain); // 根据扩展名判断文件是否存在
      }
    }

    if (!filename && rc === 1) {  // Directory. // 是一个目录
      // try it with each of the extensions at "index"
      if (exts === undefined)
        // 不存在扩展名，而获取到.js 和 .json 和 .node
        exts = ObjectKeys(Module._extensions);
      // 1. 先判断当前目录下的package.json是否存在main指定入口文件，存在则返回指定文件
      // 2. package.json不存在main则依次判断当前目录下是否存在index.js、index.json、index.node文件
      filename = tryPackage(basePath, exts, isMain, request);
    }

    if (filename) {
      // 设置缓存
      Module._pathCache[cacheKey] = filename;
      return filename;
    }
  }

  return false;
};
```



### 2、加载原生内置模块(loadNativeModule)

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/3%E3%80%81%E5%8E%9F%E7%94%9F%E6%A8%A1%E5%9D%97%E5%8A%A0%E8%BD%BD.png)



1. 从NativeModule的map对象获取到模块
2. 编译内置模块，并加载

> 源码如下

```js
// lib/internal/modules/cjs/loader.js
function loadNativeModule(filename, request) {
  // 从NativeModule获取到模块实例
  const mod = NativeModule.map.get(filename);
  if (mod?.canBeRequiredByUsers) {
    debug('load native module %s', request);
    // compileForPublicLoader() throws if mod.canBeRequiredByUsers is false:
    // 编译内置模块，并加载编译
    mod.compileForPublicLoader();
    return mod;
  }
}
```

#### （1）NativeModule类

NativeModule负责内置JS模块的加载，即编译和执行

> node.gyp配置文件有一项配置是node_js2c，作用是用python去调用一个名为tools/js2c.py的文件，内置JS模块的代码最终转成字符存在node_javascript.cc文件的，由js2c.py文件调用LoadJavaScriptSource函数加载内置js模块源码

> 源码如下

```js
// lib/internal/bootstrap/loaders.js
const {
  moduleIds,
} = internalBinding('native_module');


class NativeModule {
  /**
   * A map from the module IDs to the module instances.
   * @type {Map<string, NativeModule>}
   */
  
  // moduleIds保存了原生JS模块的名称列表，即js文件名
  // 源码中lib文件夹下的js文件(非internal文件夹)即Node自带的原生js模块
  static map = new SafeMap(
    ArrayPrototypeMap(moduleIds, (id) => [id, new NativeModule(id)])
  );

  constructor(id) {
    this.filename = `${id}.js`;
    this.id = id;
    // 不是internal开头的都可以被用户导入
    this.canBeRequiredByUsers = !StringPrototypeStartsWith(id, 'internal/');

    // The CJS exports object of the module.
    this.exports = {};
    // States used to work around circular dependencies.
    this.loaded = false;
    this.loading = false;

    // The following properties are used by the ESM implementation and only
    // initialized when the native module is loaded by users.
    /**
     * The C++ ModuleWrap binding used to interface with the ESM implementation.
     * @type {ModuleWrap|undefined}
     */
    this.module = undefined;
    /**
     * Exported names for the ESM imports.
     * @type {string[]|undefined}
     */
    this.exportKeys = undefined;
  }
  ...
}

```



#### （2）编译并加载内置模块

**js源码**

```js
// lib/internal/bootstrap/loaders.js
// Used by user-land module loaders to compile and load builtins.
compileForPublicLoader() {
    if (!this.canBeRequiredByUsers) {
      // No code because this is an assertion against bugs
      // eslint-disable-next-line no-restricted-syntax
      throw new Error(`Should not compile ${this.id} for public use`);
    }
    
    // 编译内置模块
    this.compileForInternalLoader();
    if (!this.exportKeys) {
      // When using --expose-internals, we do not want to reflect the named
      // exports from core modules as this can trigger unnecessary getters.
      const internal = StringPrototypeStartsWith(this.id, 'internal/');
      this.exportKeys = internal ? [] : ObjectKeys(this.exports);
    }
    this.getESMFacade();
    this.syncExports();
    return this.exports;
}


// lib/internal/bootstrap/loaders.js
// 编译内置模块
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
      // 标识模块已加载完成
      this.loaded = true;
    } finally {
      this.loading = false;
    }

    ArrayPrototypePush(moduleLoadList, `NativeModule ${id}`);
    return this.exports;
}
```

**c++源码**

```c++
// 对应的c++代码如下

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
  // 模块源码
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



### 3、加载模块load

**流程如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/node_source/4%E3%80%81%E5%AF%BC%E5%85%A5%E6%A8%A1%E5%9D%97.png)



1. Module._nodeModulePaths会从当前传入的文件夹开始，逐级往上查找node_modules文件夹
2. 查找文件名最长时的扩展名findLongestRegisteredExtension
3. 不同文件后缀名执行不同的解析函数（如js、json、node）
4. esm模块加载的一些支持



> 源码如下

```js
// lib/internal/modules/cjs/loader.js
// 加载模块
// Given a file name, pass it to the proper extension handler.
Module.prototype.load = function(filename) {
  debug('load %j for module %j', filename, this.id);

  assert(!this.loaded);
  this.filename = filename;
  // 找到当前文件夹的node_modules，不同系统有不同的查找逻辑
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  // 查找文件最长的扩展名
  const extension = findLongestRegisteredExtension(filename);
  // allow .mjs to be overridden
  if (StringPrototypeEndsWith(filename, '.mjs') && !Module._extensions['.mjs'])
    throw new ERR_REQUIRE_ESM(filename, true);

  // 不同文件后缀名执行不同的解析函数 如js、json、node
  Module._extensions[extension](this, filename);
  this.loaded = true; // 表示已经加载成功

  // esm模块加载的一些支持
  const esmLoader = asyncESM.esmLoader;
  // Create module entry at load time to snapshot exports correctly
  const exports = this.exports;
  // Preemptively cache
  if ((module?.module === undefined ||
       module.module.getStatus() < kEvaluated) &&
      !esmLoader.cjsCache.has(this))
    esmLoader.cjsCache.set(this, exports);
};

```

#### （1）findLongestRegisteredExtension

```js
// lib/internal/modules/cjs/loader.js
// 查找文件最长的扩展名
// Find the longest (possibly multi-dot) extension registered in
// Module._extensions
function findLongestRegisteredExtension(filename) {
  const name = path.basename(filename);
  let currentExtension;
  let index;
  let startIndex = 0;
  while ((index = StringPrototypeIndexOf(name, '.', startIndex)) !== -1) {
    startIndex = index + 1;
    if (index === 0) continue; // Skip dotfiles like .gitignore
    currentExtension = StringPrototypeSlice(name, index);
    if (Module._extensions[currentExtension]) return currentExtension;
  }
  
  // 默认js
  return '.js';
}
```

#### （2）3种不同文件对应不同的加载规则

```js
// 3种不同文件对应不同的加载规则
Module._extensions['.js'] = function(module, filename) {}
Module._extensions['.json'] = function(module, filename) {}
Module._extensions['.node'] = function(module, filename) {}
```

##### js模块的加载

1. fs.readFileSync同步读取源码内容
2. 调用module._compile函数编译并执行

> 源码如下

```js
// js模块的加载
// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  // If already analyzed the source, then it will be cached.
  const cached = cjsParseCache.get(module);
  let content;
  if (cached?.source) {
    content = cached.source;
    cached.source = undefined;
  } else {
    content = fs.readFileSync(filename, 'utf8');
  }
  if (StringPrototypeEndsWith(filename, '.js')) {
    const pkg = readPackageScope(filename);
    // Function require shouldn't be used in ES modules.
    if (pkg?.data?.type === 'module') {
      const parent = moduleParentCache.get(module);
      const parentPath = parent?.filename;
      const packageJsonPath = path.resolve(pkg.path, 'package.json');
      const usesEsm = hasEsmSyntax(content);
      const err = new ERR_REQUIRE_ESM(filename, usesEsm, parentPath,
                                      packageJsonPath);
      // Attempt to reconstruct the parent require frame.
      if (Module._cache[parentPath]) {
        let parentSource;
        try {
          // 同步读取源码内容
          parentSource = fs.readFileSync(parentPath, 'utf8');
        } catch {
          // Continue regardless of error.
        }
        if (parentSource) {
          const errLine = StringPrototypeSplit(
            StringPrototypeSlice(err.stack, StringPrototypeIndexOf(
              err.stack, '    at ')), '\n', 1)[0];
          const { 1: line, 2: col } =
              RegExpPrototypeExec(/(\d+):(\d+)\)/, errLine) || [];
          if (line && col) {
            const srcLine = StringPrototypeSplit(parentSource, '\n')[line - 1];
            const frame = `${parentPath}:${line}\n${srcLine}\n${
              StringPrototypeRepeat(' ', col - 1)}^\n`;
            setArrowMessage(err, frame);
          }
        }
      }
      throw err;
    }
  }
  // 编译源码并执行
  module._compile(content, filename);
};


```

编译并执行js源码

```js
// lib/internal/modules/cjs/loader.js
// 编译用户的源码并执行
// Run the file contents in the correct scope or sandbox. Expose
// the correct helper variables (require, module, exports) to
// the file.
// Returns exception, if any.
Module.prototype._compile = function(content, filename) {
  let moduleURL;
  let redirects;
  if (policy?.manifest) {
    moduleURL = pathToFileURL(filename);
    redirects = policy.manifest.getDependencyMapper(moduleURL);
    policy.manifest.assertIntegrity(moduleURL, content);
  }

  maybeCacheSourceMap(filename, content, this);

  // 包装类
  const compiledWrapper = wrapSafe(filename, content, this);

  let inspectorWrapper = null;
  if (getOptionValue('--inspect-brk') && process._eval == null) {
    if (!resolvedArgv) {
      // We enter the repl if we're not given a filename argument.
      if (process.argv[1]) {
        try {
          resolvedArgv = Module._resolveFilename(process.argv[1], null, false);
        } catch {
          // We only expect this codepath to be reached in the case of a
          // preloaded module (it will fail earlier with the main entry)
          assert(ArrayIsArray(getOptionValue('--require')));
        }
      } else {
        resolvedArgv = 'repl';
      }
    }

    // Set breakpoint on module start
    if (resolvedArgv && !hasPausedEntry && filename === resolvedArgv) {
      hasPausedEntry = true;
      inspectorWrapper = internalBinding('inspector').callAndPauseOnStart;
    }
  }
  const dirname = path.dirname(filename);
  const require = makeRequireFunction(this, redirects);
  let result;
  const exports = this.exports;
  const thisValue = exports;
  const module = this;
  if (requireDepth === 0) statCache = new SafeMap();
  if (inspectorWrapper) {
    result = inspectorWrapper(compiledWrapper, thisValue, exports,
                              require, module, filename, dirname);
  } else {
    // 传入参数并执行compiledWrapper
    result = ReflectApply(compiledWrapper, thisValue,
                          [exports, require, module, filename, dirname]);
  }
  hasLoadedAnyUserCJSModule = true;
  if (requireDepth === 0) statCache = null;
  return result;
};

// 包装类用户的源码
function wrapSafe(filename, content, cjsModuleInstance) {
  if (patched) {
    const wrapper = Module.wrap(content); // 生成闭包源码
    
    // 用vm编译wrapper
    return vm.runInThisContext(wrapper, {
      filename,
      lineOffset: 0,
      displayErrors: true,
      importModuleDynamically: async (specifier, _, importAssertions) => {
        const loader = asyncESM.esmLoader;
        return loader.import(specifier, normalizeReferrerURL(filename),
                             importAssertions);
      },
    });
  }
  
  try {
    return vm.compileFunction(content, [
      'exports',
      'require',
      'module',
      '__filename',
      '__dirname',
    ], {
      filename,
      importModuleDynamically(specifier, _, importAssertions) {
        const loader = asyncESM.esmLoader;
        return loader.import(specifier, normalizeReferrerURL(filename),
                             importAssertions);
      },
    });
  } catch (err) {
    if (process.mainModule === cjsModuleInstance)
      enrichCJSError(err, content);
    throw err;
  }
}
```



##### json模块的加载

直接fs.readFileSync同步读取文件的内容，然后执行JSONParse拿到JSON对象

```js
// json模块的加载
// Native extension for .json
Module._extensions['.json'] = function(module, filename) {
  const content = fs.readFileSync(filename, 'utf8');

  if (policy?.manifest) {
    const moduleURL = pathToFileURL(filename);
    policy.manifest.assertIntegrity(moduleURL, content);
  }

  try {
    // 用JSON对象格式导出文件内容
    setOwnProperty(module, 'exports', JSONParse(stripBOM(content)));
  } catch (err) {
    err.message = filename + ': ' + err.message;
    throw err;
  }
};
```



##### c++扩展模块的加载

1. 直接fs.readFileSync同步读取文件的内容

2. 调用process.dlopen方法加载.node模块

   > process.dlopen实际是c++层src/node_binding.cc文件的DLOpen函数，DLOpen又调用了libuv的uv_dlopen加载 .node 文件

```js
// c++扩展(.node)模块的加载

// Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  if (policy?.manifest) {
    const content = fs.readFileSync(filename);
    const moduleURL = pathToFileURL(filename);
    policy.manifest.assertIntegrity(moduleURL, content);
  }
  // Be aware this doesn't use `content`
  // dlopen对应的c++源码在src/node_process_methods.cc
  return process.dlopen(module, path.toNamespacedPath(filename));
};

// 为process挂载dlopen的源码
// src/node_process_methods.cc
env->SetMethod(target, "dlopen", binding::DLOpen);

// dlopen的实现
// src/node_binding.cc
// 
// DLOpen is process.dlopen(module, filename, flags).
// Used to load 'module.node' dynamically shared objects.
//
// FIXME(bnoordhuis) Not multi-context ready. TBD how to resolve the conflict
// when two contexts try to load the same shared object. Maybe have a shadow
// cache that's a plain C list or hash table that's shared across contexts?
void DLOpen(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  if (env->no_native_addons()) {
    return THROW_ERR_DLOPEN_DISABLED(
      env, "Cannot load native addon because loading addons is disabled.");
  }

  auto context = env->context();

  CHECK_NULL(thread_local_modpending);

  if (args.Length() < 2) {
    return THROW_ERR_MISSING_ARGS(
        env, "process.dlopen needs at least 2 arguments");
  }

  int32_t flags = DLib::kDefaultFlags;
  if (args.Length() > 2 && !args[2]->Int32Value(context).To(&flags)) {
    return THROW_ERR_INVALID_ARG_TYPE(env, "flag argument must be an integer.");
  }

  Local<Object> module;
  Local<Object> exports;
  Local<Value> exports_v;
  if (!args[0]->ToObject(context).ToLocal(&module) ||
      !module->Get(context, env->exports_string()).ToLocal(&exports_v) ||
      !exports_v->ToObject(context).ToLocal(&exports)) {
    return;  // Exception pending.
  }

  node::Utf8Value filename(env->isolate(), args[1]);  // Cast
  env->TryLoadAddon(*filename, flags, [&](DLib* dlib) {
    static Mutex dlib_load_mutex;
    Mutex::ScopedLock lock(dlib_load_mutex);

    // 使用lib_uv加载动态链接库，将其载入到内存uv_lib_t中
    const bool is_opened = dlib->Open(); 

    // Objects containing v14 or later modules will have registered themselves
    // on the pending list.  Activate all of them now.  At present, only one
    // module per object is supported.
    node_module* mp = thread_local_modpending; // 是在node_module_register时就赋值了
    thread_local_modpending = nullptr;

    if (!is_opened) {
      std::string errmsg = dlib->errmsg_.c_str();
      dlib->Close();
#ifdef _WIN32
      // Windows needs to add the filename into the error message
      errmsg += *filename;
#endif  // _WIN32
      THROW_ERR_DLOPEN_FAILED(env, "%s", errmsg.c_str());
      return false;
    }

    if (mp != nullptr) {
      if (mp->nm_context_register_func == nullptr) {
        if (env->force_context_aware()) {
          dlib->Close();
          THROW_ERR_NON_CONTEXT_AWARE_DISABLED(env);
          return false;
        }
      }
      // 将加载到动态链接库句柄转移到mp上
      mp->nm_dso_handle = dlib->handle_;
      
      dlib->SaveInGlobalHandleMap(mp);
    } else {
      if (auto callback = GetInitializerCallback(dlib)) {
        callback(exports, module, context);
        return true;
      } else if (auto napi_callback = GetNapiInitializerCallback(dlib)) {
        napi_module_register_by_symbol(exports, module, context, napi_callback);
        return true;
      } else {
        mp = dlib->GetSavedModuleFromGlobalHandleMap();
        if (mp == nullptr || mp->nm_context_register_func == nullptr) {
          dlib->Close();
          THROW_ERR_DLOPEN_FAILED(
              env, "Module did not self-register: '%s'.", *filename);
          return false;
        }
      }
    }

    // -1 is used for N-API modules
    if ((mp->nm_version != -1) && (mp->nm_version != NODE_MODULE_VERSION)) {
      // Even if the module did self-register, it may have done so with the
      // wrong version. We must only give up after having checked to see if it
      // has an appropriate initializer callback.
      if (auto callback = GetInitializerCallback(dlib)) {
        callback(exports, module, context);
        return true;
      }

      const int actual_nm_version = mp->nm_version;
      // NOTE: `mp` is allocated inside of the shared library's memory, calling
      // `dlclose` will deallocate it
      dlib->Close();
      THROW_ERR_DLOPEN_FAILED(
          env,
          "The module '%s'"
          "\nwas compiled against a different Node.js version using"
          "\nNODE_MODULE_VERSION %d. This version of Node.js requires"
          "\nNODE_MODULE_VERSION %d. Please try re-compiling or "
          "re-installing\nthe module (for instance, using `npm rebuild` "
          "or `npm install`).",
          *filename,
          actual_nm_version,
          NODE_MODULE_VERSION);
      return false;
    }
    CHECK_EQ(mp->nm_flags & NM_F_BUILTIN, 0);

    // Do not keep the lock while running userland addon loading code.
    Mutex::ScopedUnlock unlock(lock);
    if (mp->nm_context_register_func != nullptr) {
      // 把module和exports传给函数执行，此时为用户自己写的c++扩展中的init初始化函数
      mp->nm_context_register_func(exports, module, context, mp->nm_priv);
    } else if (mp->nm_register_func != nullptr) {
      mp->nm_register_func(exports, module, mp->nm_priv);
    } else {
      dlib->Close();
      THROW_ERR_DLOPEN_FAILED(env, "Module has no declared entry point.");
      return false;
    }

    return true;
  });

  // Tell coverity that 'handle' should not be freed when we return.
  // coverity[leaked_storage]
}

// src/node_binding.cc
// 将一个模块注册到Node的模块列表
extern "C" void node_module_register(void* m) {
  struct node_module* mp = reinterpret_cast<struct node_module*>(m);

  if (mp->nm_flags & NM_F_INTERNAL) {
    mp->nm_link = modlist_internal;
    modlist_internal = mp;
  } else if (!node_is_initialized) {
    // "Linked" modules are included as part of the node project.
    // Like builtins they are registered *before* node::Init runs.
    mp->nm_flags = NM_F_LINKED;
    mp->nm_link = modlist_linked;
    modlist_linked = mp;
  } else {
     // 把c++扩展(.node)中注册的模块赋值给thread_local_modpending
    thread_local_modpending = mp;
  }
}
```



**参考**

* https://javascript.ruanyifeng.com/nodejs/module.html#toc5

