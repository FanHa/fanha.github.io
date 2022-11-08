# 版本
+ github: envoyproxy/envoy
+ 分支:1.24.0-dev

# 序
GeneraicUpstream 是一个UpstreamRequest实例需要的结构实例,在onPoolReady时由调用方当作参数传进来,然后保存在本地uptream_变量中,UpstreamRequest通过调用upstream_ 的encodeData等方法将从下游接收到的数据交给Upstream处理

# HttpUpstream interface
```h
class HttpUpstream : public Router::GenericUpstream, public Envoy::Http::StreamCallbacks {
public:
  HttpUpstream(Router::UpstreamToDownstream& upstream_request, Envoy::Http::RequestEncoder* encoder)
      : upstream_request_(upstream_request), request_encoder_(encoder) {
    request_encoder_->getStream().addCallbacks(*this);
  }

  // GenericUpstream
  void encodeData(Buffer::Instance& data, bool end_stream) override {
    request_encoder_->encodeData(data, end_stream);
  }
  void encodeMetadata(const Envoy::Http::MetadataMapVector& metadata_map_vector) override {
    request_encoder_->encodeMetadata(metadata_map_vector);
  }
  Envoy::Http::Status encodeHeaders(const Envoy::Http::RequestHeaderMap& headers,
                                    bool end_stream) override {
    return request_encoder_->encodeHeaders(headers, end_stream);
  }
  void encodeTrailers(const Envoy::Http::RequestTrailerMap& trailers) override {
    request_encoder_->encodeTrailers(trailers);
  }

  void readDisable(bool disable) override { request_encoder_->getStream().readDisable(disable); }

  void resetStream() override {
    auto& stream = request_encoder_->getStream();
    stream.removeCallbacks(*this);
    stream.resetStream(Envoy::Http::StreamResetReason::LocalReset);
  }

  void setAccount(Buffer::BufferMemoryAccountSharedPtr account) override {
    request_encoder_->getStream().setAccount(std::move(account));
  }

  // Http::StreamCallbacks
  void onResetStream(Envoy::Http::StreamResetReason reason,
                     absl::string_view transport_failure_reason) override {
    upstream_request_.onResetStream(reason, transport_failure_reason);
  }

  void onAboveWriteBufferHighWatermark() override {
    upstream_request_.onAboveWriteBufferHighWatermark();
  }

  void onBelowWriteBufferLowWatermark() override {
    upstream_request_.onBelowWriteBufferLowWatermark();
  }

  const StreamInfo::BytesMeterSharedPtr& bytesMeter() override {
    return request_encoder_->getStream().bytesMeter();
  }

private:
  Router::UpstreamToDownstream& upstream_request_;
  Envoy::Http::RequestEncoder* request_encoder_{};
};
```
