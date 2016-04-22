用 webpack 构建 node 后端代码，使其支持 js 新特性并实现热重载  
===========================================================================

webpack 在前端领域的模块化和代码构建方面有着无比强大的功能，通过一些特殊的配置甚至可以实现前端代码的实时构建、ES6/7新特性支持以及热重载，这些功能同样可以运用于后台 nodejs 的应用，让后台的开发更加顺畅，服务更加灵活，怎么来呢？往下看。

先梳理下我们将要解决的问题：

* node端代码构建
* ES6/7 新特性支持
* node服务代码热重载

node端代码构建
-------------------
node端的代码其实是不用编译或者构建的，整个node的环境有它自己的一个模块化或者依赖机制，但是即使是现在最新的node版本，对ES6/7的支持还是捉襟见肘。当然使用一些第三方库可以做到支持类似 `async/await` 这样的语法，但是毕竟不是规范不是标准，这样看来，node端的代码还是有构建的需要的。这里我们选取的工具就是 `webpack` 以及它的一些 `loader`。   

首先，一个 `node app` 必定有一个入口文件 `app.js` ，按照 `webpack` 的规则，我们可以把所有的代码打包成一个文件 `bundle.js` ，然后运行这个 `bundle.js` 即可，`webpack.config.js` 如下：   

```js
var webpcak = require('webpack');

module.exports = {
	entry: [
	    './app.js'
	],
	output: {
	    path: path.resolve(__dirname, 'build'),
	    filename: 'bundle.js'
	}
}
```

但是有一个很严重的问题，这样打包的话，一些 `npm` 中的模块也会被打包进这个 `bundle.js`，还有 `node` 的一些原生模块，比如 `fs/path` 也会被打包进来，这明显不是我们想要的。所以我们得告诉 `webpack`，你打包的是 `node` 的代码，原生模块就不要打包了，还有 `node_modules` 目录下的模块也不要打包了，`webpack.config.js` 如下：   

```js
var webpcak = require('webpack');

var nodeModules = {};
fs.readdirSync('node_modules')
    .filter(function(x) {
        return ['.bin'].indexOf(x) === -1;
    })
    .forEach(function(mod) {
        nodeModules[mod] = 'commonjs ' + mod;
    });

module.exports = {
	entry: [
	    './app.js'
	],
	output: {
	    path: path.resolve(__dirname, 'build'),
	    filename: 'bundle.js'
	},
	target: 'node',
	externals: nodeModules
}
```

主要就是在 `webpack` 的配置中加上 `target: 'node'` 告诉 `webpack` 打包的对象是 `node` 端的代码，这样一些原生模块 `webpack` 就不会做处理。另一个就是 `webpack` 的 `externals` 属性，这个属性的主要作用就是告知 `webpack` 在打包过程中，遇到 `externals` 中声明的模块不用处理。  

比如在前端中， `jQuery` 的包通过 CDN 的方式以 `script` 标签引入，如果此时在代码中出现 `require('jQuery')` ，并且直接用 `webpack` 打包比定会报错。因为在本地并没有这样的一个模块，此时就必须在 `externals` 中声明 `jQuery` 的存在。也就是 `externals` 中的模块，虽然没有被打包，但是是代码运行是所要依赖的，而这些依赖是直接存在在整个代码运行环境中，并不用做特殊处理。  

在 `node` 端所要做的处理就是过滤出 `node_modules` 中所有模块，并且放到 `externals`中。  

这个时候我们的代码应该可以构建成功了，并且是我们期望的形态，但是不出意外的话，你还是跑不起来，因为有不小的坑存在，继续往下看。   

* 坑1：`__durname` `__filename` 指向问题
> 打包之后的代码你会发现 `__durname` `__filename` 全部都是 `/` ，这两个变量在 `webpack` 中做了一些自定义处理，如果想要正确使用，在配置中加上  
```js
context: __dirname,
node: {
    __filename: false,
    __dirname: false
},
```

* 坑2：动态 `require` 的上下文问题
>这一块比较大，放到后面讲，跟具体代码有关，和配置无关  

* 坑n：其它的还没发现，估摸不少，遇到了谷歌吧...

ES6/7 新特性支持
-------------------
构建 `node` 端代码的目标之一就是使用ES6/7中的新特性，要实现这样的目标 `babel` 是我们的不二选择。  

首先，先安装 `babel` 的各种包 `npm install babel-core babel-loader babel-plugin-transform-runtime babel-preset-es2015 babel-preset-stage-0 --save-dev json-loader -d`  

然后修改 `webpack.config.js` ，如下：

```js
var webpcak = require('webpack');

var nodeModules = {};
fs.readdirSync('node_modules')
    .filter(function(x) {
        return ['.bin'].indexOf(x) === -1;
    })
    .forEach(function(mod) {
        nodeModules[mod] = 'commonjs ' + mod;
    });

module.exports = {
	entry: [
	    './app.js'
	],
	output: {
	    path: path.resolve(__dirname, 'build'),
	    filename: 'bundle.js'
	},
	target: 'node',
	externals: nodeModules,
	context: __dirname,
	node: {
	    __filename: false,
	    __dirname: false
	},
	module: {
	    loaders: [{
	        test: /\.js$/,
	        loader: 'babel-loader',
	        exclude: [
	            path.resolve(__dirname, "node_modules"),
	        ],
	        query: {
	            plugins: ['transform-runtime'],
	            presets: ['es2015', 'stage-0'],
	        }
	    }, {
	        test: /\.json$/,
	        loader: 'json-loader'
	    }]
	},
	resolve: {
	    extensions: ['', '.js', '.json']
	}
}
```

主要就是配置 `webpack` 中的 `loader` ，借此来编译代码。  


node服务代码热重载
--------------------
`webpack` 极其牛叉的地方之一，开发的时候，实时的构建代码，并且，实时的更新你已经加载的代码，也就是说，不用手动去刷新浏览器，即可以获取最新的代码并执行。  

这一点同样可以运用在 `node` 端，实现即时修改即时生效，而不是 `pm2` 那种重启的方式。  

首先，修改配置文件，如下： 

```js
entry: [
	'webpack/hot/poll?1000',
    './app.js'
],
// ...
plugins: [
    new webpack.HotModuleReplacementPlugin()
]
```

这个时候，如果执行 `webpack --watch & node app.js` ，你的代码修改之后就可以热重载而不用重启应用，当然，代码中也要做相应改动，如下：  

```js
var hotModule = require('./hotModule');
// do something else

// 如果想要 hotModule 模块热重载
if (module.hot) {
    module.hot.accept('./hotModule.js', function() {
        var newHotModule = require('./hotModule.js');
        // do something else
    });
}
```
思路就是，如果需要某模块热重载，就把它包一层，如果修改了，`webpack` 重新打包了，重新 `require` 一遍，然后代码即是最新的代码。  

当然，如果你在某个需要热重载的模块中又依赖另一个模块，或者说动态的依赖了另一个模块，这样的模块并不会热重载。   

webpack 动态 require
-----------------------------
动态 `require` 的场景包括：  

* 场景一：在代码运行过程中遍历某个目录，动态 `reauire`，比如  

	```js
	//app.js
	var rd = require('rd');

	// 遍历路由文件夹，自动挂载路由
	var routers = rd.readFileFilterSync('./routers', /\.js/);
	routers.forEach(function(item) {
		require(item);
	})
	```

	这个时候你会发现 `'./routers'` 下的require都不是自己想要的，然后在 `bundle.js` 中找到打包之后的相应模块后，你可以看到，动态 `require` 的对象都是 `app.js` 同级目录下的 `js` 文件，而不是 `'./routers'` 文件下的 `js` 文件。为什么呢？  

	`webpack` 在打包的时候，必须把你可能依赖的文件都打包进来，并且编上号，然后在运行的时候 `require` 相应的模块 `ID` 即可，这个时候 `webpack` 获取的动态模块，就不再是你指定的目录 `'./routers'` 了，而是相对于当前文件的目录，所以，必须修正 `require` 的上下文，修改如下：  

	```js
	// 获取正确的模块
	var req = require.context("./routers", true, /\.js$/);
	var routers = rd.readFileFilterSync('./routers', /\.js/);
	routers.forEach(function(item) {
		// 使用包涵正确模块的已经被修改过的 `require` 去获取模块
		req(item);
	})
	```

* 场景二：在 `require` 的模块中含有变量，比如  

	```js
	var myModule = require(isMe ? './a.js' : './b.js');
	// 或者
	var testMoule = require('./mods' + name + '.js');
	```

	第一种的处理方式在 `webpack` 中的处理是把模块 `./a.js` `./b.js` 都包涵进来，根据变量不同 `require` 不同的模块。  

	第二种的处理方式和场景一类似，获取 `./mods/` 目录下的所有模块，然后重写了 `require` ，然后根据变量不同加载不通的模块，所以自己处理的时候方法类似。  


用 ES6/7 写 webpack.config.js
----------------------------------
项目都用 ES6/7 了，配置文件也必须跟上。  

安装好 `babel` 编译所需要的几个依赖包，然后把 `webpack.config.js` 改为 `webpack.config.babel.js` ，然后新建 `.babelrc` 的 `babel` 配置文件，加入  

```json
{
  "presets": ["es2015"]
}

```
然后和往常一样执行 `webpack` 的相关命令即可。

完整 `webpack.config.babel.js` 如下：   

```js
import webpack from 'webpack';
import fs from 'fs';
import path from 'path';

let nodeModules = {};
fs.readdirSync('node_modules')
    .filter((x) => {
        return ['.bin'].indexOf(x) === -1;
    })
    .forEach((mod) => {
        nodeModules[mod] = 'commonjs ' + mod;
    });

export default {
    cache: true,
    entry: [
        'webpack/hot/poll?1000',
        './app.js'
    ],
    output: {
        path: path.resolve(__dirname, 'build'),
        filename: 'bundle.js'
    },
    context: __dirname,
    node: {
        __filename: false,
        __dirname: false
    },
    target: 'node',
    externals: nodeModules,
    module: {
        loaders: [{
            test: /\.js$/,
            loader: 'babel-loader',
            exclude: [
                path.resolve(__dirname, "node_modules"),
            ],
            query: {
                plugins: ['transform-runtime'],
                presets: ['es2015', 'stage-0'],
            }
        }, {
            test: /\.json$/,
            loader: 'json-loader'
        }]
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin()
    ],
    resolve: {
        extensions: ['', '.js', '.json']
    }
}
```

大致流程就是如此，坑肯定还有，遇到的话手动谷歌吧～



