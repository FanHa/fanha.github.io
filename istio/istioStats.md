## 疑问
如下图所示的k8s 集群envoy 的filter配置,明明`istio.stats filter`在`local_ratelimit filter`的后面,但实际运行过程中,prom依然记录了被限流filter拒绝的请求
```json
{
  "name": "envoy.filters.http.local_ratelimit",
  "typed_config": {
    "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
    "type_url": "type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit",
    "value": {
      "filter_enabled": {
        "default_value": {
          "numerator": 100
        },
        "runtime_key": "local_rate_limit_enabled"
      },
      "filter_enforced": {
        "default_value": {
          "numerator": 100
        },
        "runtime_key": "local_rate_limit_enforced"
      },
      "response_headers_to_add": [
        {
          "append": false,
          "header": {
            "key": "x-envoy-ratelimited",
            "value": "true"
          }
        }
      ],
      "stat_prefix": "http_local_rate_limiter",
      "token_bucket": {
        "fill_interval": "0.100s",
        "max_tokens": 600,
        "tokens_per_fill": 60
      }
    }
  }
},
// ...
{
  "name": "istio.stats",
  "typed_config": {
    "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
    "type_url": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
    "value": {
      "config": {
        "configuration": {
          "@type": "type.googleapis.com/google.protobuf.StringValue",
          "value": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\",\n  \"disable_host_header_fallback\": true\n}\n"
        },
        "root_id": "stats_outbound",
        "vm_config": {
          "code": {
            "local": {
              "inline_string": "envoy.wasm.stats"
            }
          },
          "runtime": "envoy.wasm.runtime.null",
          "vm_id": "stats_outbound"
        }
      }
    }
  }
},
// ...
```
## istio 源码
`istio.stats`加载时会注册一个log回调,envoy在请求周期调用这个方法(记录日志并做其他事情,比如发给prom)
### 注册addAccessLogHandler
```cpp
// source/common/http/conn_manager_impl.cc
ttp::FilterFactoryCb IstioStatsFilterConfigFactory::createFilterFactoryFromProtoTyped(
    const stats::PluginConfig& proto_config, const std::string&,
    Server::Configuration::FactoryContext& factory_context) {
  factory_context.api().customStatNamespaces().registerStatNamespace(CustomStatNamespace);
  ConfigSharedPtr config = std::make_shared<Config>(proto_config, factory_context);
  config->recordVersion();
  return [config](Http::FilterChainFactoryCallbacks& callbacks) {
    auto filter = std::make_shared<IstioStatsFilter>(config);
    callbacks.addStreamFilter(filter);
    // Wasm filters inject filter state in access log handlers, which are called
    // after onStreamComplete.
    // 注册了一个addAccessLogHandler
    callbacks.addAccessLogHandler(filter);
  };
}
```

### 实现filter的log方法
```cpp
// source/extensions/filters/http/istio_stats/istio_stats.cc
// AccessLog::Instance
// 实现了log,envoy会在请求的周期内回调这个方法
void log(const Http::RequestHeaderMap* request_headers,
           const Http::ResponseHeaderMap* response_headers,
           const Http::ResponseTrailerMap* response_trailers, const StreamInfo::StreamInfo& info,
           AccessLog::AccessLogType) override {
    // todo
    reportHelper(true);
    // ...
           }
```
## envoy 源码相关
envoy 在`onStreamComplete`后统一调用所有filter注册了的log_handler,不适在请求经过特定filter时调用;
```cpp
// source/common/http/conn_manager_impl.cc
    stream.filter_manager_.onStreamComplete();

    // ...
    stream.filter_manager_.log(AccessLog::AccessLogType::DownstreamEnd);

```

```cpp
// source/common/http/filter_manager.h
  void log(AccessLog::AccessLogType access_log_type) {
    // ...
    // 直接在这里调用所有log_handles,而不是在请求处理经过filter时调用
    for (const auto& log_handler : access_log_handlers_) {
      log_handler->log(request_headers, response_headers, response_trailers, streamInfo(),
                       access_log_type);
    }
  }
```

## 结论
被`local_ratelimit filter`拒绝的请求虽然没有经过`istio.stats filter`,但依然会在请求的`onStreamComplete`阶段后调用`istio.stats filter`注册的log方法,从而记录日志