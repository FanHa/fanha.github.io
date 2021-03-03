## 主要结构
ChaosAgent用来和ChaosPlatform通信并调用chaosblade
![avatar](/_resource/chaosagent.png)
```go
// main.go
func main() {
	// 初始化配置
	mainConfig, err := initConfig()
    // ...

	// http传输服务,其他服务需要监听http请求,或主动使用http连接其他(如platform)时,需要用到这个服务
	httpTransport, err := transport.New(&mainConfig.TransportConfig)
	_, err = httpTransport.Start()
	
    // ...

	// 初始化日志服务
	initLog(mainConfig)

	// 心跳服务,前面生成的httpTransport作为一个参数传入;
    // 心跳服务既需要使用httpTransport监听一个/ping的http请求回应别人的心跳,也需要周期性主动向platform发送心跳
	heartbeat.New(mainConfig.HeartbeatConfig, httpTransport).Start()

	// ChaosBlade服务,同样传入httpTransport作为参数,用来监听http请求,路由转发到具体的chaosblade命令处理
	chaosBlade := chaosblade.New(httpTransport)

	// 这个Controller是为了以后的扩展方便,虽然目前只有chaosblade,但当以后有其他混沌工具(比如适应自身业务的混沌工具)时,
    // 可以用同样的写成类似Chaosblade的服务,然后注册进Controller
	ctl := controller.NewController(httpTransport)
	ctl.Register(controller.ChaosBlade, chaosBlade)
	ctl.Start()

	go func() {
		defer tools.PrintPanicStack()
        // 启动http服务(目前只有http)
		err := http.ListenAndServe(":"+mainConfig.Port, nil)
		if err != nil {
			logrus.Warningln("Start http server failed")
		}
	}()

    // 便于接到退出信号时做一些退出前的处理
	tools.Hold(ctl, httpTransport)
}
```
---
## HttpTransport
http传输服务,提供给其他服务监听http接口或使用http连接 platform
### 结构
```go
// transport/transport.go
type Transport struct {
	client   *httpClient // 公用连接platform的连接
	invoker  RequestInvoker // ??
	handlers map[string]*InterceptorRequestHandler // ??
	mutex    sync.Mutex // 公用服务必要的锁保护临界操作
	Config   *Config // 配置
}
```

### 初始化
```go
// transport/transport.go
func New(config *Config) (*Transport, error) {
	var httpClient *httpClient
    // 根据配置创建httpClient,初始化并保存platform服务的信息,便于后面使用它来连接platform服务;
    // 目前agent的启动需要指定一个的platform服务地址
	httpClient, err := NewDirectHttp(config)

    // ...

	return &Transport{
		client:   httpClient,
		invoker:  NewInvoker(httpClient, true),
		handlers: make(map[string]*InterceptorRequestHandler),
		mutex:    sync.Mutex{},
		Config:   config,
	}, nil
}

```

### 注册要监听的http路由Handler
比如心跳服务就需要注册监听一个`/ping`路由来回应platform的心跳检测
```go
// transport/transport.go
func (transport *Transport) RegisterHandler(handlerName string, handler *InterceptorRequestHandler) {
	transport.mutex.Lock()
	defer transport.mutex.Unlock()
	if transport.handlers[handlerName] == nil {
		transport.handlers[handlerName] = handler
		err := AddHttpHandler(handlerName, handler)
		// ...
	}
}

func AddHttpHandler(handlerName string, handler Handler) error {
    // 调用http库方法为 一个路由绑定传入的处理方法
	http.HandleFunc("/"+handlerName, func(writer http.ResponseWriter, request *http.Request) {
		body, err := ioutil.ReadAll(request.Body)
		// ...
		result, err := handler.Handle(string(body))
		// ...
		_, err = writer.Write([]byte(result))
		// ... 
	})
	return nil
}
```

### 启停服务
```go
// transport/transport.go
func (transport *Transport) Start() (*Transport, error) {
	err := transport.connect()
	// ...
	return transport, nil
}

// 启动服务时,需要将agent所在及其的各种信息作为一个整体汇报给platform,方便platform控制管理
func (transport *Transport) connect() error {
	request := NewRequest()
	request.AddParam("ip", meta.Info.Ip)
	request.AddParam("agentId", meta.Info.AgentId)
	request.AddParam("pid", meta.Info.Pid).AddParam("type", meta.ProgramName)
	request.AddParam("instanceId", meta.Info.InstanceId)
	request.AddParam("uid", meta.Info.Uid)
	request.AddParam("namespace", meta.Info.Namespace)
	request.AddParam("deviceId", meta.Info.InstanceId)

	request.AddParam("uptime", tools.GetUptime())
	request.AddParam("startupMode", meta.Info.StartupMode)
	request.AddParam("v", meta.Info.Version)
	request.AddParam("agentMode", meta.Info.AgentInstallMode)
	request.AddParam("cpuNum", strconv.Itoa(runtime.NumCPU()))

	if uname, err := exec.Command("uname", "-a").Output(); err != nil {
		logrus.Warnf("get os version wrong")
	} else {
		request.AddParam("osVersion", string(uname))
	}

	if memInfo, err := linux.ReadMemInfo("/proc/meminfo"); err != nil {
		logrus.Warnln("read proc/meminfo err:", err.Error())
	} else {
		memTotalKB := float64(memInfo.MemTotal)
		request.AddParam("memSize", fmt.Sprintf("%f", memTotalKB))
	}

	request.AddParam(tools.AppInstanceKeyName, meta.Info.ApplicationInstance)
	request.AddParam(tools.AppGroupKeyName, meta.Info.ApplicationGroup)

	uri := NewUri(HttpHandlerRegister)

	invoker := NewInvoker(transport.client, false)
	response, err := invoker.Invoke(uri, request)
	if err != nil {
		return err
	}

	return handleConnectResponse(*response)
}

//DoStop
func (transport *Transport) Stop() error {
    // ...
    return nil
}
```
---
## HeartBeat
用于和platform保持心跳
```go
// heartbeat/heartbeat.go
type heartbeat struct {
	period time.Duration // 周期时间
	*transport.Transport // 注入的httpTransport服务
}
```

### 初始化
```go
// heartbeat/heartbeat.go
func New(config Config, trans *transport.Transport) *heartbeat {
    // 向注入的httpTransport服务申请监听一个路由,用于响应platform服务器的心跳请求
	handler := &GetPingHandler().InterceptorRequestHandler
	trans.RegisterHandler(transport.Ping, handler)
	return &heartbeat{
		period:    config.Period,
		Transport: trans,
	}
}

```

### 启停
```go
// heartbeat/heartbeat.go
heartBeat服务启动后需要周期的向platform汇报
func (beat *heartbeat) Start() *heartbeat {
	ticker := time.NewTicker(beat.period)
	go func() {
		for range ticker.C {
			request := transport.NewRequest()

			uri := transport.NewUri(transport.HttpHandlerHeartbeat)
			if meta.IsHostMode() {
				request.AddHeader(tools.AppInstanceKeyName, meta.Info.ApplicationInstance)
				request.AddHeader(tools.AppGroupKeyName, meta.Info.ApplicationGroup)
			}
            // 发送心跳
			beat.sendHeartbeat(uri, request)
		}
	}()
    // ...
	return nil
}

// sendHeartbeat
func (beat *heartbeat) sendHeartbeat(uri transport.Uri, request *transport.Request) {
    // 这里其实调用的是前面httpTransport的Invoke方法,即向platform发送http请求
	response, err := beat.Invoke(uri, request)
	// ...
    // 备案记录自己服务的心跳历史
	beat.record(true)
}

func (beat *heartbeat) record(success bool) {
	HBSnapshotList.Put(HBSnapshot{
		Success: success,
	})
}
```
---
## ChaosBlade Service
转化platform的http请求到chaosblade的具体执行
```go
// chaosblade/blade.go
type ChaosBlade struct {
	transport *transport.Transport // 注入的httpTransport服务
	handler   *transport.InterceptorRequestHandler
	*service.Controller
	mutex   sync.Mutex
	running map[string]string
}
```

### 初始化
```go
// chaosblade/blade.go
func New(trans *transport.Transport) *ChaosBlade {
	blade := &ChaosBlade{
		transport: trans,
		running:   make(map[string]string, 0),
		mutex:     sync.Mutex{},
	}
    // 初始化blade的handler(这个handler就是接受platform的http请求后转化为chaosblade的具体执行)
	blade.handler = &GetChaosBladeHandler(blade).InterceptorRequestHandler

	blade.Controller = service.NewController(blade) // ??

    // 向httpTransport服务注册监听一个路由
	blade.transport.RegisterHandler(transport.ChaosBlade, blade.handler)
	return blade
}
```

#### chaosblade handler 的执行
```go
// chaosblade/handler.go
type Handler struct {
	transport.InterceptorRequestHandler
	blade *ChaosBlade
}

//GetFaultInjectHandler
func GetChaosBladeHandler(blade *ChaosBlade) *Handler {
	handler := &Handler{
		blade: blade,
	}
    // 用httpTransport的方法格式化好这个handler
	requestHandler := transport.NewCommonHandler(handler)
	handler.InterceptorRequestHandler = requestHandler
	return handler
}

//Handle
func (handler *Handler) Handle(request *transport.Request) *transport.Response {
	// 这里可知,chaosblade的命令是直接由platform生成好,然后作为一个整体的参数传过来直接交给chaosblade服务执行
	cmd := request.Params["cmd"]
	return handler.blade.exec(cmd)
}
```

```go
// chaosblade/blade.go
func (blade *ChaosBlade) exec(cmd string) *transport.Response {
	start := time.Now()
	fields := strings.Fields(cmd)
	// ...
	command := fields[0]
    // 执行命令
	result, errMsg, ok := tools.ExecScript(context.Background(), "/opt/chaosblade/blade", cmd)
	diffTime := time.Since(start)
    // ...
    if ok {
		response := parseResult(result)
		i// ...
        // 记录当前chaosblade命令的状态,便于后面的命令针对前面的命令的再操作
		blade.handleCacheAndSafePoint(cmd, command, fields[1], response)
		return response
	} else {
		// ...
	}
}

func (blade *ChaosBlade) handleCacheAndSafePoint(cmdline, command, arg string, response *transport.Response) {
	blade.mutex.Lock()
	defer blade.mutex.Unlock()
	if isCreateOrPrepareCmd(command) {
        // create 和 prepare 命令需要记录当前正在运行的命令id
		uid := response.Result.(string)
		blade.running[uid] = cmdline
	} else if isDestroyOrRevokeCmd(command) {
        // destory 和 revoke 命令,将指定命令id的状态取消掉
		var uid = arg
		if _, ok := blade.running[uid]; ok {
			delete(blade.running, uid)
		}
		if isRevokeOperation(command) {
			record, err := blade.queryPreparationStatus(uid)
			// ...
		}
	}
}
```

### 启停
chaosblade服务的是由另一个服务Controller来启停的(和前面的heartbeat等服务不同),需要实现的接口是DoStart 和 DoStop(供Controller服务调用)
```go
// chaosblade/blade.go
func (blade *ChaosBlade) DoStart() error {
	blade.handler.Start()
	return nil
}

// chaosblade 服务停止时需要遍历正在运行中的chaosblade命令,作出相应的“反操作
func (blade *ChaosBlade) DoStop() error {
	blade.mutex.Lock()
	var copyRunning = make(map[string]string, 0)
	for key, value := range blade.running {
		copyRunning[key] = value
	}
	blade.mutex.Unlock()
    // 遍历正在运行中的chaosblade命令,作出“反操作”
	for k, v := range copyRunning {
		var response *transport.Response
		command := strings.Fields(v)[0]
		if _, ok := createOperation[command]; ok {
			response = blade.exec(fmt.Sprintf("destroy %s", k))
		}
		if _, ok := prepareOperation[command]; ok {
			response = blade.exec(fmt.Sprintf("revoke %s", k))
		}
		if !response.Success {
			logrus.Errorf("!!stop %s command err: %s", v, response.Error)
		}
	}
	return blade.handler.Stop()
}
```
---
## Controller服务
Controller服务包裹了Chaosblade服务,同时也向其他潜在的别的混沌工具提供了注册接口
```go
// controller/controller.go
type Controller struct {
	services   map[string]service.LifeCycle // 以注册的服务,目前只有chaosblade
	serviceKey []string // 服务key
	transport  *transport.Transport // 注入的httpTransport服务,用来处理http请求
	handler    *transport.InterceptorRequestHandler
	mutex      sync.Mutex
	*service.Controller // ??
	shutdownFuncList []func() error
}
```

### 初始化Controller
```go
//NewController
func NewController(transport0 *transport.Transport) *Controller {
	control := &Controller{
		services:         make(map[string]service.LifeCycle, 0),
		serviceKey:       make([]string, 0),
		transport:        transport0,
		shutdownFuncList: make([]func() error, 0),
	}
	control.Controller = service.NewController(control)
	control.handler = &GetControllerHandler(control).InterceptorRequestHandler
	return control
}
```

### 注册服务到controller
```go
// controller/controller.go
func (controller *Controller) Register(serviceName string, service service.LifeCycle) {
	controller.mutex.Lock()
	defer controller.mutex.Unlock()
	if controller.services[serviceName] == nil { // 重复注册判断
		controller.serviceKey = append(controller.serviceKey, serviceName) // 将服务加入到服务列表
		controller.services[serviceName] = service
		logrus.Infof("[Controller] register %s service to controller", serviceName)
	}
}
```

### 启停Controller
```go
// controller/controller.go
func (controller *Controller) DoStart() error {
	go func() {
		controller.mutex.Lock()
		defer controller.mutex.Unlock()
        // 遍历所有已经注册的服务,调用他们的Start方法
		for _, key := range controller.serviceKey {
			lifeCycle := controller.services[key]
			if lifeCycle != nil {
				err := lifeCycle.Start()
				if err != nil {
					logrus.Warningf("[Controller] start %s service failed, err: %s", key, err.Error())
					continue
				}
				logrus.Infof("[Controller] start %s service successfully.", key)
			}
		}
	}()
	return nil
}
func (controller *Controller) DoStop() error {
	go func() {
		defer tools.PrintPanicStack()
		controller.mutex.Lock()
		defer controller.mutex.Unlock()
		length := len(controller.serviceKey)
        // 遍历所有已经注册的服务,调用他们的Stop方法
		for i := length - 1; i >= 0; i-- {
			key := controller.serviceKey[i]
			lifeCycle := controller.services[key]
			if lifeCycle != nil {
				err := lifeCycle.Stop()
				if err != nil {
					logrus.Warningf("[Controller] stop %s service failed, err: %s", key, err.Error())
					continue
				}
				logrus.Infof("[Controller] stop %s service successfully.", key)
			}
		}
	}()
	return nil
}
```

