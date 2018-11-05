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
  // 启动
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
      if (typeof options.before === 'function') {
        options.before(app, this);
      }
    },
    // 前面生成的wabpack-compiler-middleware就是在这里作为一个中间件传入express
    middleware: () => {
      // include our middleware to ensure it is able to handle '/index.html' request after redirect
      app.use(this.middleware);
    },
    after: () => {
      if (typeof options.after === 'function') { options.after(app, this); }
    },
    headers: () => {
      app.all('*', this.setContentHeaders.bind(this));
    },
    magicHtml: () => {
      app.get('*', this.serveMagicHtml.bind(this));
    },
    setup: () => {
      if (typeof options.setup === 'function') {
        log('The `setup` option is deprecated and will be removed in v3. Please update your config to use `before`');
        options.setup(app, this);
      }
    }
  };

  // 根据配置决定是否启用feature中的处理机制
  (options.features || defaultFeatures).forEach((feature) => {
    features[feature]();
  });

  //启动http服务器
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
`webpack-dev-server` 通过生成一个`webpack-dev-middleware`中间件 传入 express,httpDev服务器收到http请求时会把请求交给这个中间件处理;
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
	var shared = Shared(context);

	// The middleware function
	function webpackDevMiddleware(req, res, next) {
		function goNext() {
			if(!context.options.serverSideRender) return next();
			return new Promise(function(resolve) {
				shared.ready(function() {
					res.locals.webpackStats = context.webpackStats;
					resolve(next());
				}, req);
			});
		}

		if(req.method !== "GET") {
			return goNext();
		}

		var filename = getFilenameFromUrl(context.options.publicPath, context.compiler, req.url);
		if(filename === false) return goNext();

		return new Promise(function(resolve) {
			shared.handleRequest(filename, processRequest, req);
			function processRequest() {
				try {
					var stat = context.fs.statSync(filename);
					if(!stat.isFile()) {
						if(stat.isDirectory()) {
							var index = context.options.index;

							if(index === undefined || index === true) {
								index = "index.html";
							} else if(!index) {
								throw "next";
							}

							filename = pathJoin(filename, index);
							stat = context.fs.statSync(filename);
							if(!stat.isFile()) throw "next";
						} else {
							throw "next";
						}
					}
				} catch(e) {
					return resolve(goNext());
				}

				// server content
				var content = context.fs.readFileSync(filename);
				content = shared.handleRangeHeaders(content, req, res);
				var contentType = mime.lookup(filename);
				// do not add charset to WebAssembly files, otherwise compileStreaming will fail in the client
				if(!/\.wasm$/.test(filename)) {
					contentType += "; charset=UTF-8";
				}
				res.setHeader("Content-Type", contentType);
				res.setHeader("Content-Length", content.length);
				if(context.options.headers) {
					for(var name in context.options.headers) {
						res.setHeader(name, context.options.headers[name]);
					}
				}
				// Express automatically sets the statusCode to 200, but not all servers do (Koa).
				res.statusCode = res.statusCode || 200;
				if(res.send) res.send(content);
				else res.end(content);
				resolve();
			}
		});
	}

	webpackDevMiddleware.getFilenameFromUrl = getFilenameFromUrl.bind(this, context.options.publicPath, context.compiler);
	webpackDevMiddleware.waitUntilValid = shared.waitUntilValid;
	webpackDevMiddleware.invalidate = shared.invalidate;
	webpackDevMiddleware.close = shared.close;
	webpackDevMiddleware.fileSystem = context.fs;
	return webpackDevMiddleware;
};
```