### 入口
开发服务器时一般用 npm run dev 启动一个开发服务器,实际调用的是webpack-dev-server
```json
{
    // ...
    "script": {
        "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    }
    // ...
}
```

### webpack-dev-server
```js
// webpack-dev-server/bin/webpack-dev-server.js

// ...
// 解析配置文件生成wpOpt,然后传入processOptions
processOptions(wpOpt);
```

```js
// bin/webpack-dev-server.js
function processOptions(webpackOptions) {

  /*
  * 把不同方式定义的参数合并(命令行传参,配置文件传参,默认参等)
  */
  const options = webpackOptions.devServer || firstWpOpt.devServer || {};
  // ...
  
  portfinder.basePort = DEFAULT_PORT;
  portfinder.getPort((err, port) => {
    if (err) throw err;
    options.port = port;
    // 将合并后的参数传入startDevServer
    startDevServer(webpackOptions, options);
  });
}
```

```js
// webpack-dev-server/bin/webpack-dev-server.js
function startDevServer(webpackOptions, options) {
  // ...
  let compiler;
  try {
    // 生成了一个webpack的compiler,用来抽象所有服务器请求的处理
    compiler = webpack(webpackOptions);
  } catch (e) {
    // ...
  }

  let server;
  try {
    // 新建一个服务器实例,传入前面生成的webpack-compiler;
    // 这里的Server对 express 的一层封装,
    // 新建一个express实例,做一些配置,返回该express实例
    server = new Server(compiler, options);
  } catch (e) {
    //...
  }
  //...
  // 启动(并附带启动了一个websocket服务)
  server.listen(options.socket, options.host, (err) => {
    //...
  });
}

```

### Server
```js
// webpack-dev-server/lib/Server.js
function Server(compiler, options) {
  if (!options) options = {};

  // Init express server
  const app = this.app = new express(); 

  app.all('*', (req, res, next) => {
    if (this.checkHost(req.headers)) { return next(); }
    res.send('Invalid Host header');
  });

  const wdmOptions = {};

  // 用webpack-compiler生成一个中间件
  this.middleware = webpackDevMiddleware(compiler, Object.assign({}, options, wdmOptions));

  // 一些与业务内容无关的开发模式接口
  app.get('/__webpack_dev_server__/live.bundle.js', (req, res) => {
    // ...
  });
  app.get('/__webpack_dev_server__/sockjs.bundle.js', (req, res) => {
    // ...
  });
  app.get('/webpack-dev-server.js', (req, res) => {
    // ...
  });
  app.get('/webpack-dev-server/*', (req, res) => {
    // ...
  });
  app.get('/webpack-dev-server', (req, res) => {
    //...
  });

  // webpack-dev-server的一些常见组件features,后面会通过查看配置来觉得是否调用这些features,
  const features = {
    compress() {
      // ...
    },
    proxy() {
      // ...
    },
    historyApiFallback() {
      // ...
    },
    contentBaseFiles() {
      // ...
    },
    contentBaseIndex() {
      // ...
    },
    watchContentBase: () => {
      // ...
    },
    before: () => {
      // ...
    },
    // 前面生成的wabpack-compiler-middleware就是在这里作为一个中间件传入express
    middleware: () => {
      // include our middleware to ensure it is able to handle '/index.html' request after redirect
      app.use(this.middleware);
    },
    after: () => {
      // ...
    },
    headers: () => {
      // ...
    },
    magicHtml: () => {
      // ...
    },
    setup: () => {
      // ...
    }
  };

  // 根据配置决定是否启用feature中的处理机制
  (options.features || defaultFeatures).forEach((feature) => {
    features[feature]();
  });

  //启动http(s)服务器
  if (options.https) {
    this.listeningApp = spdy.createServer(options.https, app);
  } else {
    this.listeningApp = http.createServer(app);
  }

  // TODO 这个地方的作用???
  websocketProxies.forEach(function (wsProxy) {
    this.listeningApp.on('upgrade', wsProxy.upgrade);
  }, this);
}
```

### webpack-dev-middleware
+ `webpack-dev-server` 通过生成一个`webpack-dev-middleware`中间件(this.middleware);
+ 然后通过`app.use(this.middleware)` 传入express;
+ httpDev服务器收到http请求时会把请求交给这个中间件处理;

middleware中间件内容如下
```js
// webpack-dev-middleware/middleware.js
module.exports = function(compiler, options) {

	var context = {
		state: false,
		webpackStats: undefined,
		callbacks: [],
		options: options,
		compiler: compiler,
		watching: undefined,
		forceRebuild: false
	};

  // 初始化环境,使context.fs不直接访问文件系统,而交给compiler(即webpack解析器)
	var shared = Shared(context);

	// 这个函数就是要返回给express做回调的函数
	function webpackDevMiddleware(req, res, next) {

		var filename = getFilenameFromUrl(context.options.publicPath, context.compiler, req.url);

		return new Promise(function(resolve) {
      // 这里懒加载,需要让compiler检查文件状态,预先准备好文件的内容,然后后面才能使用context.fs.readFileSync(filename)
			shared.handleRequest(filename, processRequest, req);
			function processRequest() {
				try {

					var stat = context.fs.statSync(filename);
					//...
				} catch(e) {
					return resolve(goNext());
				}

				// 读取要get的文件
				var content = context.fs.readFileSync(filename);
				
        // 返回结果
				if(res.send) res.send(content);
				else res.end(content);
				resolve();
			}
		});
	}

  //...
	return webpackDevMiddleware;
};
```

 + 中间件通过Shared方法把访问文件的文件系统指向了一个新生成的`MemoryFileSystem`;
 + 然后通过在读取文件前,先调用`handleRequest`方法,让`compiler`按需准备好文件;
 + 再读文件,返回给客户端;
```js
// webpack-dev-middleware/lib/Shared.js
module.exports = function Shared(context) {
	var share = {
		setFs: function(compiler) {
			
			fs = compiler.outputFileSystem = new MemoryFileSystem();
			context.fs = fs;
		},
    // ...
		handleRequest: function(filename, processRequest, req) {
			// in lazy mode, rebuild on bundle request
			if(context.options.lazy && (!context.options.filename || context.options.filename.test(filename)))
        //这个里面会调用compiler来生成虚拟的文件
				share.rebuild();
			if(HASH_REGEXP.test(filename)) {
				try {
					if(context.fs.statSync(filename).isFile()) {
						processRequest();
						return;
					}
				} catch(e) {
				}
			}
			share.ready(processRequest, req);
		},
    rebuild: function rebuild() {
			if(context.state) {
				context.state = false;
				context.compiler.run(share.handleCompilerCallback);
			} else {
				context.forceRebuild = true;
			}
		},
	};
  // 设置context的文件系统,使读取文件从webpackCompiler生成的MemoryFileSystem里读
	share.setFs(context.compiler);
	return share;
};
```

### 开发时服务器怎样通知前端页面访问的文件发生了变化,以便让前端js做出相应的操作(刷新页面)
服务在`startDevServer`阶段,调用`listen`方法时
```js
// webpack-dev-server/lib/Server.js
Server.prototype.listen = function (port, hostname, fn) {

  const returnValue = this.listeningApp.listen(port, hostname, (err) => {

    // sockjs时对websocket的一层封装,兼容了一些不能使用websocket机制的浏览器
    const sockServer = sockjs.createServer({
      //...
    });
    sockServer.on('connection', (conn) => {
      // 收到客户端的连接后将该连接push到sockets中,后面有什么事件触发时用来发送消息给客户端(如开服时改了服务端代码,通知浏览器,浏览器收到通知后刷新页面)
      this.sockets.push(conn);
    });
    // 客户端请求建立连接的入口
    sockServer.installHandlers(this.listeningApp, {
      prefix: '/sockjs-node'
    });
  });

  return returnValue;
}

```

```js
// webpack-dev-server/lib/Server.js
function Server(compiler, options) {
  // ...
  // 新建Server实例时,向compiler(webpack)注册监听了事件done,即webpack重新编译好开发的改动后,会触发;
  compiler.plugin('done', (stats) => {
    this._sendStats(this.sockets, stats.toJson(clientStats));
    this._stats = stats;
  });
  // ...
}
Server.prototype._sendStats = function (sockets, stats, force) {
  // 发送给所有连接此服服务器的开发的浏览器
  this.sockWrite(sockets, 'hash', stats.hash);

};
```