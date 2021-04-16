
.. _config_http_filters_aws_lambda:

AWS Lambda
==========

* :ref:`v3 API 参考 <envoy_v3_api_msg_extensions.filters.http.aws_lambda.v3.Config>`
* 过滤器的名称应该被配置为 *envoy.filters.http.aws_lambda*。

.. attention::

  AWS Lambda 过滤器目前正在积极开发中。

HTTP AWS Lambda 过滤器用于从标准 HTTP/1.x 或 HTTP/2 请求中触发 AWS Lambda 函数。

如果 :ref:`payload_passthrough <envoy_v3_api_field_extensions.filters.http.aws_lambda.v3.Config.payload_passthrough>` 被设置为 ``true``，然后将负载发送到 Lambda，而不进行任何转换。

但如果 :ref:`payload_passthrough <envoy_v3_api_field_extensions.filters.http.aws_lambda.v3.Config.payload_passthrough>` 被设置为 ``false``，HTTP 请求被转换为具有以下模式的 JSON 负载：

.. code-block::

    {
        "rawPath": "/path/to/resource",
        "method": "GET|POST|HEAD|...",
        "headers": {"header-key": "header-value", ... },
        "queryStringParameters": {"key": "value", ...},
        "body": "...",
        "isBase64Encoded": true|false
    }

- ``rawPath`` 是 HTTP 请求支援路径（包含查询字符串）
- ``method`` 是 HTTP 请求方法。例如 ``GET``, ``PUT`` 等。
- ``headers`` 是 HTTP 请求头。如果多个头使用相同的名称，则他们的值会合并为一个逗号分隔的值。
- ``queryStringParameters`` 是 HTTP 请求查询字符串参数。如果多个参数使用相同的名称，最后一个会将之前的替换。也就是说，如果参数共享同一个键名称，则它们不会合并为同一个值。
- ``body`` 如果 ``content-type`` 头存在并且不是下面之一，则 HTTP 请求的请求体由过滤器进行 base64 编码：

    -  text/*
    -  application/json
    -  application/xml
    -  application/javascript

Otherwise, the body of HTTP request is added to the JSON payload as is.
否则，HTTP 请求的请求体会原样添加到 JSON 负载中。

另一方面，Lambda 函数的响应必须符合以下模式：

.. code-block::

    {
        "statusCode": ...
        "headers": {"header-key": "header-value", ... },
        "cookies": ["key1=value1; HttpOnly; ...", "key2=value2; Secure; ...", ...],
        "body": "...",
        "isBase64Encoded": true|false
    }

- ``statusCode`` 字段是用作 HTTP 响应码的整数。如果这个键消失，Envoy 会返回一个 ``200 OK``。
-  ``headers`` 用作 HTTP 响应头。
- ``cookies`` 用作 ``Set-Cookie`` 响应头。与请求头不一样，cookies 不是响应头的一部分，因为每个 `RFC` 的 ``Set-Cookie`` 头不包含多个值。因此，每一个在 JSON 数组中的键值对会被翻译为单个 ``Set-Cookie`` 头。
- 如果被标记为 base64 编码并且发送的是 HTTP 响应体，则对 ``body`` 进行 base64 解码。

.. _RFC: https://tools.ietf.org/html/rfc6265#section-4.1

.. note::

    The target cluster must have its endpoint set to the `regional Lambda endpoint`_. Use the same region as the Lambda
    function.
    目标集群必须将其端点设置为 `区域 Lambda 端点`。使用与 Lambda 函数相同的区域。

    AWS IAM credentials must be defined in either environment variables, EC2 metadata or ECS task metadata.
    AWS IAM 凭据必须被定义在环境变量、EC2 元数据或 ECS 任务元数据中。


.. _regional Lambda endpoint: https://docs.aws.amazon.com/general/latest/gr/lambda-service.html

过滤器支持 :ref:`每个过滤器配置
<envoy_v3_api_msg_extensions.filters.http.aws_lambda.v3.PerRouteConfig>`.

If you use the per-filter configuration, the target cluster _must_ have the following metadata:
如果你使用每个过滤器配置，则目标集群必须包含以下元数据：

.. code-block:: yaml

    metadata:
      filter_metadata:
        com.amazonaws.lambda:
          egress_gateway: true


Below are some examples that show how the filter can be used in different deployment scenarios.
下面有一些示例，展示如何在不同的部署场景中使用过滤器。

配置示例
---------------------

In this configuration, the filter applies to all routes in the filter chain of the http connection manager:
在这个配置中，过滤器在 HTTP 连接管理器过滤器链中应用所有路由：

.. code-block:: yaml

  http_filters:
  - name: envoy.filters.http.aws_lambda
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.aws_lambda.v3.Config
      arn: "arn:aws:lambda:us-west-2:987654321:function:hello_envoy"
      payload_passthrough: true

The corresponding regional endpoint must be specified in the target cluster. So, for example if the Lambda function is
in us-west-2:
必须在目标集群中指定相应的区域终结点。例如，如果 Lambda 函数在 us-west-2 中：

.. code-block:: yaml

  clusters:
  - name: lambda_egress_gateway
    connect_timeout: 0.25s
    type: LOGICAL_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: lambda_egress_gateway
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: lambda.us-west-2.amazonaws.com
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: "*.amazonaws.com"


The filter can also be configured per virtual-host, route or weighted-cluster. In that case, the target cluster *must*
have specific Lambda metadata.
还可以为每个虚拟主机、路由和权重集群配置过滤器。在这种情况下，目标集群 *必须* 具备特定的 Lambda 元数据。

.. code-block:: yaml

    weighted_clusters:
    clusters:
    - name: lambda_egress_gateway
      weight: 42
      typed_per_filter_config:
        envoy.filters.http.aws_lambda:
          "@type": type.googleapis.com/envoy.extensions.filters.http.aws_lambda.v3.PerRouteConfig
          invoke_config:
            arn: "arn:aws:lambda:us-west-2:987654321:function:hello_envoy"
            payload_passthrough: false


An example with the Lambda metadata applied to a weighted-cluster:
将 Lambda 元数据应用于权重集群的示例：

.. code-block:: yaml

  clusters:
  - name: lambda_egress_gateway
    connect_timeout: 0.25s
    type: LOGICAL_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    metadata:
      filter_metadata:
        com.amazonaws.lambda:
          egress_gateway: true
    load_assignment:
      cluster_name: lambda_egress_gateway # does this have to match? seems redundant
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: lambda.us-west-2.amazonaws.com
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: "*.amazonaws.com"


统计信息
----------

The AWS Lambda filter outputs statistics in the *http.<stat_prefix>.aws_lambda.* namespace. The
:ref:`stat prefix <envoy_api_field_config.filter.network.http_connection_manager.v2.HttpConnectionManager.stat_prefix>`
comes from the owning HTTP connection manager.
AWS Lambda 过滤器在 *http.<stat_prefix>.aws_lambda.* 命名空间中输出统计信息。
:ref:`stat 前置 <envoy_api_field_config.filter.network.http_connection_manager.v2.HttpConnectionManager.stat_prefix>` 来自于拥有的 HTTP 连接管理器。

.. csv-table::
  :header: 名称, 类型, 描述
  :widths: 1, 1, 2

  server_error, Counter, 返回无效 JSON 响应的请求总数（看 :ref:`payload_passthrough <envoy_api_msg_config.filter.http.aws_lambda.v2alpha.config>` ）。
  upstream_rq_payload_size, Histogram, JSON 转换后请求的字节大小（如果有）。

