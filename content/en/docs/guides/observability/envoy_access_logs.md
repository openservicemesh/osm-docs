---
title: "Envoy Access Logs"
description: "Configuring Envoy access logs"
type: docs
weight: 3
---

# Envoy Access Logs

Envoy proxies log access information to their standard output. The access logs can be observed by viewing the Envoy sidecar container's logs. OSM allows configuring the access log format per pod, namespace, or globally throughout the mesh. Additionally, OSM allows exporting the access logs from one or more pods in the mesh to an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) to receive and process telemetry data in a centralized manner.


## Configuring Envoy access log format

Envoy access logs contain information about various attributes of the traffic proxied through Envoy. OSM enables a default set of fields that are logged as a part of Envoy’s access logging capability in JSON format. For e.g, access logs for HTTP traffic may look as follows:

```json
{"bytes_sent":0,"request_id":"7feeda0a-31d6-4da1-b046-c5f0a1614255","protocol":"HTTP/1.1","method":"HEAD","time_to_first_byte":504,"start_time":"2022-09-08T16:41:31.933Z","response_code_details":"via_upstream","user_agent":"curl/7.85.0-DEV","upstream_host":"127.0.0.1:14001","bytes_received":0,"response_flags":"-","requested_server_name":"httpbin.httpbin.svc.cluster.local","x_forwarded_for":null,"response_code":200,"upstream_service_time":"409","path":"/","duration":508,"authority":"httpbin.httpbin:14001","upstream_cluster":"httpbin/httpbin|14001|local"}
```

OSM allows configuring the access log format per pod, namespace, or globally throughout the mesh. Additionally, the access log can be configured to be exported to a centralized OpenTelemetry Collector service. OSM leverages its [Telemetry API](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.Telemetry) to configure access logging.

The Telemetry API allows configuring access logging at 3 levels of granularity within the mesh, with the most specific configuration superseding less specific ones in case multiple configurations are applicable to an Envoy instance. The scope of the configuration is defined below in the order of specificity.

1. Per pod(s): Similar to the behavior of the [Kubernetes Service selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#service-and-replicationcontroller), the `spec.selector` field can be set to select the pods in the resource’s namespace the configuration applies to.

2. Per namespace: Configuration is applied in a namespace without the `spec.selector` field. All pods in the namespace inherit the configuration.

3. Global: Configuration is applied without the `spec.selector` field in the OSM root namespace (osm-system by default).


### Formats

The access log format is specified using the `accessLog.format` field. It can either be a textual representation or a JSON string representation of the access log format string. Refer to the [Envoy access log documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#access-logging) to learn more about [access log format strings](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#format-strings) and the [command operators](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators) that can be used to extract values inserted into the access logs.

To generate an access log of the form `{"authority":"httpbin.httpbin:14001","bytes_sent":0,"bytes_received":0}` for the `httpbin` app in the `test` namespace, the configuration will look like:

```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: Telemetry
metadata:
  name: httpbin
  namespace: test
spec:
  selector:
    app: httpbin
  accessLog:
    format: '{"authority":"%REQ(:AUTHORITY)%","bytes_received":"%BYTES_RECEIVED%","bytes_sent":"%BYTES_SENT%"}'
```

To generate an access log of the form `[2022-10-19T20:23:53.392Z] authority=httpbin.httpbin:14001 bytes_received=0 bytes_sent=0` for all apps in the `test` namespace, the configuration will look like:

```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: Telemetry
metadata:
  name: test-log
  namespace: test
spec:
  accessLog:
    format: "[%START_TIME%] authority=%REQ(:AUTHORITY)% bytes_received=%BYTES_RECEIVED% bytes_sent=%BYTES_SENT%\n"
```

## Exporting access logs to an OpenTelemetry Collector

The access logs can be configured to be exported to an OpenTelemetry Collector using the [Telemetry](/docs/api_reference/policy/v1alpha1/#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting) and [ExtensionService](/docs/api_reference/config/v1alpha2/#config.openservicemesh.io/v1alpha2.ExtensionService) APIs. The Telemetry configuration specifies the pods for which access logs are to be exported and a reference to the OpenTelemetry Collector's ExtensionService configuration. The ExtensionService configuration specifies information regarding the OpenTelemetry Collector service.

For example, to configure and export access logs for the `httpbin` app to an OpenTelemetry Collector service with the address `otel-collector.otel.svc.cluster.local` and port `4317`

```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: Telemetry
metadata:
  name: httpbin-log
  namespace: httpbin
spec:
  selector:
    app: httpbin

  accessLog:
    format: "[%START_TIME%] authority=%REQ(:AUTHORITY)% bytes_received=%BYTES_RECEIVED% bytes_sent=%BYTES_SENT%\n"

    openTelemetry:
      extensionService:
        name: otel-collector
        namespace: otel
---
apiVersion: config.openservicemesh.io/v1alpha2
kind: ExtensionService
metadata:
  name: otel-collector
  namespace: otel
spec:
  host: otel-collector.otel.svc.cluster.local
  port: 4317
  protocol: h2c
```
> Note: OSM currently only supports exporting the logs to the OpenTelemetry Collector over an unencrypted gRPC connection. The `protocol` field in the ExtensionService must be set to `h2c`.

The above configuration will produce an access log similar to the below snipper on the Envoy sidecar of the `httpbin` pod:
```
[2022-10-20T15:53:13.764Z] authority=httpbin.httpbin:14001 bytes_received=0 bytes_sent=0
```

Additionally, since an OpenTelemetry configuration is specified, the access log will be exported to the OpenTelemetry Collector service. The access log message will be set in the [Body](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/logs/v1/logs.proto#L151) field of the OpenTelemetry log record as seen below in the OpenTelemetry Collector's logs:

```
2022-10-20T15:53:14.945Z	info	LogsExporter	{"kind": "exporter", "data_type": "logs", "name": "logging", "#logs": 1}
2022-10-20T15:53:14.948Z	info	ResourceLog #0
Resource SchemaURL:
Resource labels:
     -> log_name: STRING(otel-collector.otel.svc.cluster.local.4317)
     -> zone_name: STRING()
     -> cluster_name: STRING(httpbin.httpbin)
     -> node_name: STRING(fabd4869-ce46-421e-a81b-977ebbc3c9b5)
ScopeLogs #0
ScopeLogs SchemaURL:
InstrumentationScope
LogRecord #0
ObservedTimestamp: 1970-01-01 00:00:00 +0000 UTC
Timestamp: 2022-10-20 15:53:13.764598 +0000 UTC
Severity:
Body: [2022-10-20T15:53:13.764Z] authority=httpbin.httpbin:14001 bytes_received=0 bytes_sent=0

Trace ID:
Span ID:
Flags: 0
	{"kind": "exporter", "data_type": "logs", "name": "logging"}
```

OSM allows configuring additional [attributes](https://github.com/open-telemetry/opentelemetry-proto/blob/main/opentelemetry/proto/logs/v1/logs.proto#L156) as metadata for the access logs exported to the OpenTelemetry Collector. This can be done by setting the `spec.accessLogs.openTelemetry.attributes` field.

For example, the following configuration will produce the attribute `somekey: someValue` to be logged as attributes in the OpenTelemetry log record:
```yaml
apiVersion: policy.openservicemesh.io/v1alpha1
kind: Telemetry
metadata:
  name: httpbin-log
  namespace: httpbin
spec:
  selector:
    app: httpbin

  accessLog:
    format: "[%START_TIME%] authority=%REQ(:AUTHORITY)% bytes_received=%BYTES_RECEIVED% bytes_sent=%BYTES_SENT%\n"

    openTelemetry:
      extensionService:
        name: otel-collector
        namespace: otel
      attributes:
        someKey: "someValue"
---
apiVersion: config.openservicemesh.io/v1alpha2
kind: ExtensionService
metadata:
  name: otel-collector
  namespace: otel
spec:
  host: otel-collector.otel.svc.cluster.local
  port: 4317
  protocol: h2c
```

```
2022-10-20T16:00:25.428Z	info	LogsExporter	{"kind": "exporter", "data_type": "logs", "name": "logging", "#logs": 1}
2022-10-20T16:00:25.428Z	info	ResourceLog #0
Resource SchemaURL:
Resource labels:
     -> log_name: STRING(otel-collector.otel.svc.cluster.local.4317)
     -> zone_name: STRING()
     -> cluster_name: STRING(httpbin.httpbin)
     -> node_name: STRING(fabd4869-ce46-421e-a81b-977ebbc3c9b5)
ScopeLogs #0
ScopeLogs SchemaURL:
InstrumentationScope
LogRecord #0
ObservedTimestamp: 1970-01-01 00:00:00 +0000 UTC
Timestamp: 2022-10-20 16:00:24.382057 +0000 UTC
Severity:
Body: [2022-10-20T16:00:24.382Z] authority=httpbin.httpbin:14001 bytes_received=0 bytes_sent=0

Attributes:
     -> someKey: STRING(someValue)
Trace ID:
Span ID:
Flags: 0
	{"kind": "exporter", "data_type": "logs", "name": "logging"}
```