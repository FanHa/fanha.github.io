# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
UpstreamRequest 在 http filter的最后一个filter router中被生成并使用,用来将请求发送给上游

# 关键interface
## acceptHeadersFromRouter
在router-filter对请求头经过处理后,生成了‘UpstreamRequest’实例,并调用‘acceptHeadersFromRouter’方法
```cpp
// source/common/router/upstream_request.cc
void UpstreamRequest::acceptHeadersFromRouter(bool end_stream) {
  ASSERT(!router_sent_end_stream_);
  router_sent_end_stream_ = end_stream;

  // Kick off creation of the upstream connection immediately upon receiving headers.
  // In future it may be possible for upstream filters to delay this, or influence connection
  // creation but for now optimize for minimal latency and fetch the connection
  // as soon as possible.
  conn_pool_->newStream(this);
  if (!allow_upstream_filters_) {
    return;
  }

  auto* headers = parent_.downstreamHeaders();

  // Make sure that when we are forwarding CONNECT payload we do not do so until
  // the upstream has accepted the CONNECT request.
  if (headers->getMethodValue() == Http::Headers::get().MethodValues.Connect) {
    paused_for_connect_ = true;
  }

  filter_manager_->requestHeadersInitialized();
  filter_manager_->streamInfo().setRequestHeaders(*parent_.downstreamHeaders());
  filter_manager_->decodeHeaders(*parent_.downstreamHeaders(), end_stream);
}
```
