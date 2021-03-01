.. _why_is_envoy_sending_http2_resets:

为什么 Envoy 要发送 HTTP/2 重置？
===================================

HTTP/2 重置路径主要由 Envoy 编解码器使用 HTTP/2、nghttp2 来控制。
nghttp2 对于 HTTP/2 有着非常好的遵从性，但是由于许多客户端并不完全遵从，因此这种不完全匹配就会导致以外的重置。
不幸的是，与调试 :ref:`内部响应路径 <why_is_envoy_sending_internal_responses>` 不同，Envoy 在给定具体原因的 nghttp2 中重置流有着可限的能见度。

如果你有一个可再现的失败案例，你可以在带有 "-l trace" 的 Envoy 中调试运行它，那么你将获得详细的 nghttp2 异常日志，这些日志通常会指出哪些头信息未通过符合性检查。
或者，如果你也可以在一台遇到异常的机器上带有 "-l trace" 运行，那么你可以从格式为 `invalid http2: [nghttp2 error detail]` 的文件 "source/common/http/http2/codec_impl.cc" 中找到日志。
例如：`invalid http2: Invalid HTTP header field was received: frame type: 1, stream: 1, name: [content-length], value: [3]`

另外你也可以检查 :ref:`HTTP/2 状态`<config_http_conn_man_stats_per_codec>`: 在许多情况下，Envoy 会重置流。
例如如果有更多的头信息超过配置的允许界限或流检查点启用，那么 http2 计数器将会增加，这时流也会被重置。


