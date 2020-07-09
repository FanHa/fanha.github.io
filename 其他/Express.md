### 使用方
+ 调用方通过`var app = express();`来初始化一个express;
+ 配置express服务器的各种行为,如路由,中间件等;
+ 调用`listen`方法启动服务器;
```js
var express = require('express');
var app = express();

app.get('/', function (req, res) {
    //...
});
app.listen(3000, function () {
    //...
});

```

### 生成`express`实例
```js
// express.js
exports = module.exports = createApplication;

function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };

  //混入事件处理方法,让app支持'on', 'emit'等方法
  mixin(app, EventEmitter.prototype, false);

  //混入基本的调用方法,如上面使用方使用的'get','listen', 和下面初始化过程时的'init'
  mixin(app, proto, false);

  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // 初始化一些内部变量和存储结构
  app.init();
  return app;
}
```

### 路由使用方法
调用方使用如`get`,`post`等HTTP方法生成基本的路由和处理,传入一个路由路径参数`'/'`, 和一个回调`function(req,res) ...`
```js
// GET method route
app.get('/', function (req, res) {
  res.send('GET request to the homepage');
});

// POST method route
app.post('/', function (req, res) {
  res.send('POST request to the homepage');
});
```

在app混入的proto方法里统一循环了所有HTTP方法生成了如`app.get`,`app.post`等方法
```js
//application.js
methods.forEach(function(method){
  app[method] = function(path){

    //懒加载生成一个路由器Router
    this.lazyrouter();

    //给路由器新加入一条路由(route),以及触发该路由时的回调,即前面传入的function(req,res)
    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});

app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```

### 启动服务

```js
//application.js
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```
+ 调用了node.js 的标准库方法`http.createServer()`来生成一个http服务,并listen参数传入的端口;
+ 注意这里的`this`指向的是前面的
```js
var app = function(req, res, next) {
    app.handle(req, res, next);
  };
```
+ 因此,所有发往http服务器的请求的处理都封装在`app.handle()`处理后面;

### app.handle
```js
app.handle = function handle(req, res, callback) {
  var router = this._router;

  // final handler
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  // no routes
  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }

  // 逻辑处理转到Router的handle方法,这样用户可以对Router进行编程,实现不同的路由转不同的处理逻辑,和中间件执行链等
  router.handle(req, res, done);
};
```

### 路由,中间件
路由主要有 `Router.use()` 和 `Router.handle()`
#### Router.use
前面懒加载路由时
```js
// application.js
app.lazyrouter = function lazyrouter() {
  this._router.use(query(this.get('query parser fn')));
  this._router.use(middleware.init(this));
}
```

```js
// router/index.js
proto.use = function use(fn) {
  // ...
  var callbacks = flatten(slice.call(arguments, offset));

  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];

    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    // 这里stack是Router的公共变量,Router通过多次调用use()方法,将请求的处理回调压入公共变量stack,这样可以use不同的回调件依次处理请求
    this.stack.push(layer);
  }

  return this;
};
```

#### Router.handle
前面use了对请求处理的middleware后,请求的handle阶段,handle 通过while循环遍历stack,并根据配置的路径(route)和回调进行相应处理
```js
proto.handle = function handle(req, res, out) {
  var self = this;

  // middleware and routes
  var stack = self.stack;
  // setup next layer
  req.next = next;

  next();

  function next(err) {
    while (match !== true && idx < stack.length) {
      layer = stack[idx++];
      match = matchLayer(layer, path);
      route = layer.route;

      var method = req.method;
      var has_method = route._handles_method(method);

    }

    // store route for dispatch on change
    if (route) {
      req.route = route;
    }

    // this should be done for the layer
    self.process_params(layer, paramcalled, req, res, function (err) {
      if (route) {
        return layer.handle_request(req, res, next);
      }
    });
  }
};
```


