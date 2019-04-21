---
title: ESL内部实现
date: 2016-10-13 10:35:24
---

## ESL (Enterprise Standard Loader)

> ESL是一个浏览器端、符合AMD的标准加载器，适合用于现代Web浏览器端应用的入口与模块管理。

``` js
require(['echarts', 'echarts/chart/line', 'echarts/chart/pie'], dosomething);
```

上面代码的执行流程是：

1. require创建三个 script 标签'echarts', 'echarts/chart/line', 'echarts/chart/pie'
2. 在上面动态创建的 script 标签中， 注册 onload 事件
3. onload 方法的实现，则是判断这个三个标签，是否全部加载完毕，当然，必然存在一个文件是最后被加载的，所以这个最后被加载的文件将调用 dosomething 方法。

	``` js
    function tryFinishRequire() {
        if (typeof callback === 'function' && !isCallbackCalled) {
            var isAllCompleted = 1;
            each(ids, function (id) {
                if (!BUILDIN_MODULE[id]) {
                    return (isAllCompleted = !!modIs(id, MODULE_DEFINED));
                }
            });

            // 检测并调用callback
            if (isAllCompleted) {
                isCallbackCalled = 1;

                callback.apply(
                    global,
                    modGetModulesExports(ids, BUILDIN_MODULE)
                );
            }
        }
    }
	```

[ecomfe/esl](https://github.com/ecomfe/esl/)

类型功能的库：[RequireJS](http://requirejs.org/)

[requirejs/requirejs](https://github.com/requirejs/requirejs/)

## require

使用 esl 的 require 可以实现动态加载 js ,例如：

``` js
// 在这里可以定义多个 package
require.config({
		packages: [{
				name: 'echarts',
				location: '${pageContext.request.contextPath}/js/echarts',	  
				main: 'echarts'
			},{
				name: 'zrender',
				location: '${pageContext.request.contextPath}/js/zrender', 
				main: 'zrender'
			}]
});

// 第一个参数指定，从哪个 package 中查找 location
require(['echarts', 'echarts/chart/line'], showLineChart);
```

[Common-Config](hhttps://github.com/amdjs/amdjs-api/blob/master/CommonConfig.md)

[require(Array, Function)](https://github.com/amdjs/amdjs-api/blob/master/require.md)

[ESL配置](https://github.com/ecomfe/esl/blob/master/doc/config.md)

## why require?

* 将js引入到项目中的全局变量进行了屏蔽，js 引入的功能通过变量引入 showLineChart，而不会污染整个页面中的全局变量。

	其实这里也存在一个问题，例如 jquery 将导出 $ 和 jQuery 这两个全局变量，如果使用这种方式，加载jquery,将 $ 的功能限制于，其回调函数，这是非常不方便的。

* 便于升级 require 所加载的库，因为具体的 js 都是由 require 的 packages 来描述的，如果需要升级则修改 location 即可。

	对于一个项目来说，通常只需要一个 require.config 即可，所以可以将 require.config 的单独放置到一个 js 当中。每一个具体的页面只需要引用这个js即可，这样当需要升级js时可以直接修改那一个js文件即可。

## 内部实现

``` js
// 2.1.4
// 121 行
function globalRequire(requireId, callback) {
}

// 82
 var actualGlobalRequire = createLocalRequire();
 
// 1224
// 在这个函数中进行package的解析
    function createLocalRequire(baseId) {

// 765
   function nativeAsyncRequire(ids, callback, baseId) {

// 821
    function loadModule(moduleId) {
    
// 1574
    function createScript(moduleId, onload) {
```

### 异步下载

使用 `document.createElement` 创建一个 script 标签。然后前 src 设置成需要下载的script文件的路径。然后调用 `appendChild` 将 script 元素添加成 head 标签的子元素。由于设置了 `script.async = true`, 所以浏览器将执行异步下载。

``` js
// 这个方法被 loadModule 方法调用用来加载 script 文件。
function createScript(moduleId, onload) {

    // 1. 创建script标签
    var script = document.createElement('script');
    script.setAttribute('data-require-id', moduleId);
    script.src = toUrl(moduleId + '.js');
    script.async = true;
    
    // 2. 设置回调
    if (script.readyState) {
        script.onreadystatechange = innerOnload;
    }
    else {
        script.onload = innerOnload;
    }

	// 3. script 文件加载并执行完毕之后执行的回调函数
    function innerOnload() {
        var readyState = script.readyState;
        if (
            typeof readyState === 'undefined'
            || /^(loaded|complete)$/.test(readyState)
        ) {
            script.onload = script.onreadystatechange = null;
            script = null;

            onload();
        }
    }
    
    // ---------------------------------------
    // 4. !!!加载脚本!!!
    // currentlyAddingScript 变量用来跟踪当前正在添加的script文件
    currentlyAddingScript = script;

	// 将 新创建的 script 元素添加到 <head></head>
	// headElement 代表 head 标签，注意如果文档中没有 head 标签
	// 则可能出现问题:
	// var headElement = document.getElementsByTagName('head')[0];
	// appendChild 方法瘵新创建的 script 元素添加到 head 中。
	// 此时user agent(浏览器)将下载 script 文件。
	// 由于有 script.async 的设置，所以文件的下载将异步执行。
	// 而不会阻塞当前的线程的执行。
    baseElement
        ? headElement.insertBefore(script, baseElement)
        : headElement.appendChild(script);

	// 由于 script 已经添加成功，所以将这个变量置为 null
    currentlyAddingScript = null;
    // ---------------------------------------
}
```

### 模块缓存

``` js
/**
 * 正在加载的模块列表
 */
var loadingModules = {};

/**
 * @param {string} moduleId 模块标识
 */
function loadModule(moduleId) {
    // 加载过的模块，就不要再继续了
    if (loadingModules[moduleId] || modModules[moduleId]) {
        return;
    }
    
    // 加载文件前将 moduleId 放置到 loadingModules 对象中，
    // 当下次还出现这个 moduleId 的时候，就可以直接返回了。
    loadingModules[moduleId] = 1;
    
    // ...
    
    // 发送请求去加载模块
    function load() {
        /* eslint-disable no-use-before-define */
        var bundleModuleId = bundlesIndex[moduleId];
        createScript(bundleModuleId || moduleId, loaded);
        /* eslint-enable no-use-before-define */
    }

    // script标签加载完成的事件处理函数
    // 当 script 加载完毕之后，下面的处理按照 AMD
    // 进行模块的注册，其实就是记录模块的基本信息
    function loaded() {
    
    	// model 加载完毕之后，进行定义
    	// ==> loaded
    	// ==> modAutoDefine
    	// ==> modUpdatePreparedState
    	// ==> modPrepare
    	// ==> modInitFactoryInvoker
    	// ==> modDefined(调用回调)
        modCompletePreDefine(moduleId);

		// 
        modAutoDefine();
    }
}

// 回调函数的注册
// require(['echarts', 'echarts/chart/line'], dosomething);
function nativeAsyncRequire(ids, callback, baseId) {
    var isCallbackCalled = 0;

    each(ids, function (id) {
        if (!(BUILDIN_MODULE[id] || modIs(id, MODULE_DEFINED))) {
        	// modAddDefinedListener 方法将 tryFinishRequire 这个回调
        	// 和上面的 ['echarts', 'echarts/chart/line'] 的两个模块
        	// 关联起来，这样当这几个模块中的最后一个被加载的模块
        	// 成功加载完毕之后 tryFinishRequire 就会执行 callback
            modAddDefinedListener(id, tryFinishRequire);
            (id.indexOf('!') > 0
                ? loadResource
                : loadModule
            )(id, baseId);
        }
    });
    tryFinishRequire();

    /**
     * 尝试完成require，调用callback
     * 在模块与其依赖模块都加载完时调用
     */
    function tryFinishRequire() {
        if (typeof callback === 'function' && !isCallbackCalled) {
            var isAllCompleted = 1;
            // 检测所有的模块是否全部加载完毕
            each(ids, function (id) {
                if (!BUILDIN_MODULE[id]) {
                    return (isAllCompleted = !!modIs(id, MODULE_DEFINED));
                }
            });

            // 检测并调用callback
            if (isAllCompleted) {
                isCallbackCalled = 1;

                callback.apply(
                    global,
                    modGetModulesExports(ids, BUILDIN_MODULE)
                );
            }
        }
    }
}

// 描述一个模块的数据结构
// 模块容器
var modModules = {};
// 模块内部信息包括
// -----------------------------------
// id: module id
// depsDec: 模块定义时声明的依赖
// deps: 模块依赖，默认为['require', 'exports', 'module']
// factory: 初始化函数或对象
// factoryDeps: 初始化函数的参数依赖
// exports: 模块的实际暴露对象（AMD定义）
// config: 用于获取模块配置信息的函数（AMD定义）
// state: 模块当前状态
// require: local require函数
// depMs: 实际依赖的模块集合，数组形式
// depMkv: 实际依赖的模块集合，表形式，便于查找
// depRs: 实际依赖的资源集合
// ------------------------------------
modModules[id] = {
    id         : id,
    depsDec    : dependencies,
    deps       : dependencies || ['require', 'exports', 'module'],
    factoryDeps: [],
    factory    : factory,
    exports    : {},
    config     : moduleConfigGetter,
    state      : MODULE_PRE_DEFINED,
    require    : createLocalRequire(id),
    depMs      : [],
    depMkv     : {},
    depRs      : [],
    hang       : 0
};
```

## 执行回调

## require 对名字的解析

调用 `toUrl` 方法进行名称解析

``` js
// toUrl(moduleId + '.js');
// 所以 source 参数的形式一般是 'moduleId.js' 
function toUrl(source, baseId) {
    // 分离 模块标识 和 .extension
    
    // 扩展名正则：扩展名可以是 [a-z0-9]+ 中的多个字符
    // 同时由于使用了 i ，所以是不区分大小写的
    // 12.js, abc.JS, qwe.Js 都可以匹配到。 
    var extReg = /(\.[a-z0-9]+)$/i;
    
    // URL参数正则：url中可能带有参数，#部分代表网页中的一个位置
    // 其自身和http请求没有任何关系。
    // echarts/chart/line?param=value#location1.js
    // 下面的正则，将取出参数部分 ?param=value
    var queryReg = /(\?[^#]*)$/;
    
    // 用来存储提取出的扩展名
    var extname = '';
    // 用来存储提取出的id
    var id = source;
    // 用来存储提取出的参数
    var query = '';

    if (queryReg.test(source)) {
    	// RegExp.$1 表示上面匹配到的第一个分组
    	// 也就是 query = '?param=value' 
        query = RegExp.$1;
        // 去掉 source 中的查询参数
        source = source.replace(queryReg, '');
    }

    if (extReg.test(source)) {
    	// extname = '.js'
        extname = RegExp.$1;
        // 去掉 source 中的后缀名
        // 此时 id = 'echarts/chart/line'
        id = source.replace(extReg, '');
    }

    if (baseId != null) {
        id = normalize(id, baseId);
    }

	// echarts/chart/line
    var url = id;

    // paths处理和匹配
    // path 匹配和 packages 是互斥的。
    // 如果 id 和 pathsIndex 中的某个 item 中的某个 key 
    // 匹配成功，则将其替换成 path 中对应的 value.
    var isPathMap;
    indexRetrieve(id, pathsIndex, function (value, key) {
    	// 其实 key 和 url 应该是相同的，
    	// 所以下面的转换相当于 url = value
        url = url.replace(key, value);
        isPathMap = 1;
    });

    // packages处理和匹配
    // 如果 id 和  packagesIndex 中的某个 key(pkg.name) 匹配成功，则
    // 将 url 配置成  item.location。
    if (!isPathMap) {
        indexRetrieve(id, packagesIndex, function (value, key, item) {
            url = url.replace(item.name, item.location);
        });
    }

    // 相对路径时，附加baseUrl
    if (!/^([a-z]{2,10}:\/)?\//i.test(url)) {
        url = requireConf.baseUrl + url;
    }

    // 附加 .extension 和 query
    url += extname + query;

    // urlArgs处理和匹配
    indexRetrieve(id, urlArgsIndex, function (value) {
        url += (url.indexOf('?') > 0 ? '&' : '?') + value;
    });

    return url;
}
```

``` js
require(['echarts', 'echarts/chart/line'], dosomething);

// 其中id参数就是 'echarts' 和 'echarts/chart/line'
function (id, i) {

    var idInfo = parseId(id);
    var absId = normalize(idInfo.mod, baseId);
    var resId = idInfo.res;
    var normalizedId = absId;

    if (resId) {
        var trueResId = absId + '!' + resId;
        if (resId.indexOf('.') !== 0 && bundlesIndex[trueResId]) {
            absId = normalizedId = trueResId;
        }
        else {
            normalizedId = null;
        }
    }

    normalizedIds[i] = normalizedId;
    modFlagAutoDefine(absId);
    pureModules.push(absId);
}

function parseId(id) {
    var segs = id.split('!');

    if (segs[0]) {
        return {
            mod: segs[0],
            res: segs[1]
        };
    }
}
```

## require 对象的初始化

``` js
/**
 * require配置
 *
 * @inner
 * @type {Object}
 */
var requireConf = {
    baseUrl    : './',
    paths      : {},
    config     : {},
    map        : {},
    packages   : [],
    shim       : {},
    // #begin-ignore
    waitSeconds: 0,
    // #end-ignore
    bundles    : {},
    urlArgs    : {}
};

// -----
paths:{
	echartsPath:'echarts/chart/line',
	zrenderPath:'zrender'
},
packages: [
	{
		name: 'echarts',
		location: 'echarts',	  
		main: 'echarts'
	},
	{
		name: 'zrender',
		location: 'zrender/src',
		main: 'zrender'
	}
]

// 这些配置项，都会在 require.config 方法中，转换成数组：

paths ==> pathsIndex = [{k:'echartsPath',v:'echarts/chart/line',reg:'^<k>(\/|$)'},{k:'zrenderPath',v:'zrender',reg:'^<k>(\/|$)'}]

// -----
// package 的转换方法
// 1. package 以字符串形式提供
var pkg = packageConf;
if (typeof packageConf === 'string') {
    pkg = {
        name: packageConf.split('/')[0],
        location: packageConf,
        main: 'main'
    };
}

pkg.location = pkg.location || pkg.name;
// pkg.main 默认为 'main', 如果 pkg.main 中带有 .js
// 后缀，则将其去掉
pkg.main = (pkg.main || 'main').replace(/\.js$/i, '');
// pkg.reg = '^<pkg.name>(\/|$)'
pkg.reg = createPrefixRegexp(pkg.name);

// 这个方法就是我们调用的 require.config 方法
// 用来初始化 requireConf 对象。
globalRequire.config = function (conf) {
    if (conf) {
        for (var key in requireConf) {
            var newValue = conf[key];
            var oldValue = requireConf[key];

            if (!newValue) {
                continue;
            }

            if (key === 'urlArgs' && typeof newValue === 'string') {
                requireConf.urlArgs['*'] = newValue;
            }
            else {
                // 简单的多处配置还是需要支持，所以配置实现为支持二级mix
                if (oldValue instanceof Array) {
                    oldValue.push.apply(oldValue, newValue);
                }
                else if (typeof oldValue === 'object') {
                    for (var k in newValue) {
                        oldValue[k] = newValue[k];
                    }
                }
                else {
                    requireConf[key] = newValue;
                }
            }
        }

        createConfIndex();
    }
};

```

## 浏览器解析html及script标签

[[WebKit]WebCore之页面加载的设计与实现(一)](http://blog.csdn.net/horkychen/article/details/8888428)

[How WebKit Loads a Web Page](https://webkit.org/blog/1188/how-webkit-loads-a-web-page/)

## 参考

* [\<script\> async Attribute](http://www.w3schools.com/tags/att_script_async.asp)
* [\<script\> defer Attribute](http://www.w3schools.com/tags/att_script_defer.asp)
* [JavaScript 的性能优化：加载和执行](https://www.ibm.com/developerworks/cn/web/1308_caiys_jsload/)
* [Script标签和脚本执行顺序](http://pij.robinqu.me/Browser_Scripting/Document_Loading/ScriptTag.html)
* [HTML Scripting](https://w3c.github.io/html/semantics-scripting.html)
* [element-attrdef-script-async](https://w3c.github.io/html/semantics-scripting.html#element-attrdef-script-async)
* [The Chromium Projects](http://dev.chromium.org/)
* [chromium源码](https://src.chromium.org/viewvc)
* [Git repositories on chromium](https://chromium.googlesource.com/)
* [require.js](https://github.com/requirejs/requirejs/blob/master/require.js)
* [esl.js](https://github.com/ecomfe/esl/blob/master/src/esl.js)
* [Document Object Model](https://www.w3.org/DOM/)
* [amdjs-api](https://github.com/amdjs/amdjs-api)
* [Asynchronous Module Definition (AMD)](https://github.com/amdjs/amdjs-api/blob/master/AMD.md)
* [config 配置](http://requirejs.org/docs/api.html)