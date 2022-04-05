---
title: "Config v1alpha2 API Reference"
description: "Config v1alpha2 API reference documentation."
type: docs
---

<p>Packages:</p>
<ul>
<li>
<a href="#config.openservicemesh.io%2fv1alpha2">config.openservicemesh.io/v1alpha2</a>
</li>
</ul>
<h2 id="config.openservicemesh.io/v1alpha2">config.openservicemesh.io/v1alpha2</h2>
<div>
<p>Package v1alpha2 is the v1alpha2 version of the API.</p>
</div>
Resource Types:
<ul></ul>
<h3 id="config.openservicemesh.io/v1alpha2.CertificateSpec">CertificateSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<div>
<p>CertificateSpec is the type to reperesent OSM&rsquo;s certificate management configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>serviceCertValidityDuration</code><br/>
<em>
string
</em>
</td>
<td>
<p>ServiceCertValidityDuration defines the service certificate validity duration.</p>
</td>
</tr>
<tr>
<td>
<code>certKeyBitSize</code><br/>
<em>
int
</em>
</td>
<td>
<p>CertKeyBitSize defines the certicate key bit size.</p>
</td>
</tr>
<tr>
<td>
<code>ingressGateway</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.IngressGatewayCertSpec">
IngressGatewayCertSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>IngressGateway defines the certificate specification for an ingress gateway.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ClusterSpec">ClusterSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MultiClusterServiceSpec">MultiClusterServiceSpec</a>)
</p>
<div>
<p>ClusterSpec is the type used to represent a remote cluster in multicluster scenarios.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>address</code><br/>
<em>
string
</em>
</td>
<td>
<p>Address defines the remote IP address of the gateway</p>
</td>
</tr>
<tr>
<td>
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name defines the name of the remote cluster.</p>
</td>
</tr>
<tr>
<td>
<code>weight</code><br/>
<em>
int
</em>
</td>
<td>
<p>Weight defines the load balancing weight of the remote cluster</p>
</td>
</tr>
<tr>
<td>
<code>priority</code><br/>
<em>
int
</em>
</td>
<td>
<p>Priority defines the priority of the remote cluster in locality based load balancing</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ExternalAuthzSpec">ExternalAuthzSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.TrafficSpec">TrafficSpec</a>)
</p>
<div>
<p>ExternalAuthzSpec is a type to represent external authorization configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>enable</code><br/>
<em>
bool
</em>
</td>
<td>
<p>Enable defines a boolean indicating if the external authorization policy is to be enabled.</p>
</td>
</tr>
<tr>
<td>
<code>address</code><br/>
<em>
string
</em>
</td>
<td>
<p>Address defines the remote address of the external authorization endpoint.</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
uint16
</em>
</td>
<td>
<p>Port defines the destination port of the remote external authorization endpoint.</p>
</td>
</tr>
<tr>
<td>
<code>statPrefix</code><br/>
<em>
string
</em>
</td>
<td>
<p>StatPrefix defines a prefix for the stats sink for this external authorization policy.</p>
</td>
</tr>
<tr>
<td>
<code>timeout</code><br/>
<em>
string
</em>
</td>
<td>
<p>Timeout defines the timeout in which a response from the external authorization endpoint.
is expected to execute.</p>
</td>
</tr>
<tr>
<td>
<code>failureModeAllow</code><br/>
<em>
bool
</em>
</td>
<td>
<p>FailureModeAllow defines a boolean indicating if traffic should be allowed on a failure to get a
response against the external authorization endpoint.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.FeatureFlags">FeatureFlags
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<div>
<p>FeatureFlags is a type to represent OSM&rsquo;s feature flags.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>enableWASMStats</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableWASMStats defines if WASM Stats are enabled.</p>
</td>
</tr>
<tr>
<td>
<code>enableEgressPolicy</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableEgressPolicy defines if OSM&rsquo;s Egress policy is enabled.</p>
</td>
</tr>
<tr>
<td>
<code>enableMulticlusterMode</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableMulticlusterMode defines if Multicluster mode is enabled.</p>
</td>
</tr>
<tr>
<td>
<code>enableSnapshotCacheMode</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableSnapshotCacheMode defines if XDS server starts with snapshot cache.</p>
</td>
</tr>
<tr>
<td>
<code>enableAsyncProxyServiceMapping</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableAsyncProxyServiceMapping defines if OSM will map proxies to services asynchronously.</p>
</td>
</tr>
<tr>
<td>
<code>enableIngressBackendPolicy</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableIngressBackendPolicy defines if OSM will use the IngressBackend API to allow ingress traffic to
service mesh backends.</p>
</td>
</tr>
<tr>
<td>
<code>enableEnvoyActiveHealthChecks</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableEnvoyActiveHealthChecks defines if OSM will Envoy active health
checks between services allowed to communicate.</p>
</td>
</tr>
<tr>
<td>
<code>enableRetryPolicy</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableRetryPolicy defines if retry policy is enabled.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.IngressGatewayCertSpec">IngressGatewayCertSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.CertificateSpec">CertificateSpec</a>)
</p>
<div>
<p>IngressGatewayCertSpec is the type to represent the certificate specification for an ingress gateway.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>subjectAltNames</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>SubjectAltNames defines the Subject Alternative Names (domain names and IP addresses) secured by the certificate.</p>
</td>
</tr>
<tr>
<td>
<code>validityDuration</code><br/>
<em>
string
</em>
</td>
<td>
<p>ValidityDuration defines the validity duration of the certificate.</p>
</td>
</tr>
<tr>
<td>
<code>secret</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#secretreference-v1-core">
Kubernetes core/v1.SecretReference
</a>
</em>
</td>
<td>
<p>Secret defines the secret in which the certificate is stored.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshConfig">MeshConfig
</h3>
<div>
<p>MeshConfig is the type used to represent the mesh configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>metadata</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Object&rsquo;s metadata.</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">
MeshConfigSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the MeshConfig specification.</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>sidecar</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.SidecarSpec">
SidecarSpec
</a>
</em>
</td>
<td>
<p>Sidecar defines the configurations of the proxy sidecar in a mesh.</p>
</td>
</tr>
<tr>
<td>
<code>traffic</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.TrafficSpec">
TrafficSpec
</a>
</em>
</td>
<td>
<p>Traffic defines the traffic management configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>observability</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ObservabilitySpec">
ObservabilitySpec
</a>
</em>
</td>
<td>
<p>Observalility defines the observability configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>certificate</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.CertificateSpec">
CertificateSpec
</a>
</em>
</td>
<td>
<p>Certificate defines the certificate management configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>featureFlags</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.FeatureFlags">
FeatureFlags
</a>
</em>
</td>
<td>
<p>FeatureFlags defines the feature flags for a mesh instance.</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfig">MeshConfig</a>)
</p>
<div>
<p>MeshConfigSpec is the spec for OSM&rsquo;s configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>sidecar</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.SidecarSpec">
SidecarSpec
</a>
</em>
</td>
<td>
<p>Sidecar defines the configurations of the proxy sidecar in a mesh.</p>
</td>
</tr>
<tr>
<td>
<code>traffic</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.TrafficSpec">
TrafficSpec
</a>
</em>
</td>
<td>
<p>Traffic defines the traffic management configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>observability</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ObservabilitySpec">
ObservabilitySpec
</a>
</em>
</td>
<td>
<p>Observalility defines the observability configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>certificate</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.CertificateSpec">
CertificateSpec
</a>
</em>
</td>
<td>
<p>Certificate defines the certificate management configurations for a mesh instance.</p>
</td>
</tr>
<tr>
<td>
<code>featureFlags</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.FeatureFlags">
FeatureFlags
</a>
</em>
</td>
<td>
<p>FeatureFlags defines the feature flags for a mesh instance.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MultiClusterService">MultiClusterService
</h3>
<div>
<p>MultiClusterService is the type used to represent the multicluster configuration.
MultiClusterService name needs to match the name of the service backing the pods in each cluster.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>metadata</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#objectmeta-v1-meta">
Kubernetes meta/v1.ObjectMeta
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Object&rsquo;s metadata.</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MultiClusterServiceSpec">
MultiClusterServiceSpec
</a>
</em>
</td>
<td>
<p>Spec is the MultiClusterService specification.</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>clusters</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ClusterSpec">
[]ClusterSpec
</a>
</em>
</td>
<td>
<p>ClusterSpec defines the configuration of other clusters</p>
</td>
</tr>
<tr>
<td>
<code>serviceAccount</code><br/>
<em>
string
</em>
</td>
<td>
<p>ServiceAccount represents the service account of the multicluster service.</p>
</td>
</tr>
<tr>
<td>
<code>ports</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>Ports is the list of ports exported by this service.</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MultiClusterServiceSpec">MultiClusterServiceSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MultiClusterService">MultiClusterService</a>)
</p>
<div>
<p>MultiClusterServiceSpec is the type used to represent the multicluster service specification.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>clusters</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ClusterSpec">
[]ClusterSpec
</a>
</em>
</td>
<td>
<p>ClusterSpec defines the configuration of other clusters</p>
</td>
</tr>
<tr>
<td>
<code>serviceAccount</code><br/>
<em>
string
</em>
</td>
<td>
<p>ServiceAccount represents the service account of the multicluster service.</p>
</td>
</tr>
<tr>
<td>
<code>ports</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>Ports is the list of ports exported by this service.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ObservabilitySpec">ObservabilitySpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<div>
<p>ObservabilitySpec is the type to represent OSM&rsquo;s observability configurations.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>osmLogLevel</code><br/>
<em>
string
</em>
</td>
<td>
<p>OSMLogLevel defines the log level for OSM control plane logs.</p>
</td>
</tr>
<tr>
<td>
<code>enableDebugServer</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableDebugServer defines if the debug endpoint on the OSM controller pod is enabled.</p>
</td>
</tr>
<tr>
<td>
<code>tracing</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.TracingSpec">
TracingSpec
</a>
</em>
</td>
<td>
<p>Tracing defines OSM&rsquo;s tracing configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.PortSpec">PortSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MultiClusterServiceSpec">MultiClusterServiceSpec</a>)
</p>
<div>
<p>PortSpec contains information on service&rsquo;s port.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>Port</code><br/>
<em>
uint32
</em>
</td>
<td>
<p>The port that will be exposed by this service.</p>
</td>
</tr>
<tr>
<td>
<code>Protocol</code><br/>
<em>
string
</em>
</td>
<td>
<p>Protocol is The IP protocol for this port. Supports &ldquo;TCP&rdquo;, &ldquo;UDP&rdquo;, and &ldquo;SCTP&rdquo;. Default is TCP.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.SidecarSpec">SidecarSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<div>
<p>SidecarSpec is the type used to represent the specifications for the proxy sidecar.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>enablePrivilegedInitContainer</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnablePrivilegedInitContainer defines a boolean indicating whether the init container for a meshed pod should run as privileged.</p>
</td>
</tr>
<tr>
<td>
<code>logLevel</code><br/>
<em>
string
</em>
</td>
<td>
<p>LogLevel defines the logging level for the sidecar&rsquo;s logs. Non developers should generally never set this value. In production environments the LogLevel should be set to error.</p>
</td>
</tr>
<tr>
<td>
<code>envoyImage</code><br/>
<em>
string
</em>
</td>
<td>
<p>EnvoyImage defines the container image used for the Envoy proxy sidecar.</p>
</td>
</tr>
<tr>
<td>
<code>envoyWindowsImage</code><br/>
<em>
string
</em>
</td>
<td>
<p>EnvoyWindowsImage defines the windows container image used for the Envoy proxy sidecar.</p>
</td>
</tr>
<tr>
<td>
<code>initContainerImage</code><br/>
<em>
string
</em>
</td>
<td>
<p>InitContainerImage defines the container image used for the init container injected to meshed pods.</p>
</td>
</tr>
<tr>
<td>
<code>maxDataPlaneConnections</code><br/>
<em>
int
</em>
</td>
<td>
<p>MaxDataPlaneConnections defines the maximum allowed data plane connections from a proxy sidecar to the OSM controller.</p>
</td>
</tr>
<tr>
<td>
<code>configResyncInterval</code><br/>
<em>
string
</em>
</td>
<td>
<p>ConfigResyncInterval defines the resync interval for regular proxy broadcast updates.</p>
</td>
</tr>
<tr>
<td>
<code>resources</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#resourcerequirements-v1-core">
Kubernetes core/v1.ResourceRequirements
</a>
</em>
</td>
<td>
<p>Resources defines the compute resources for the sidecar.</p>
</td>
</tr>
<tr>
<td>
<code>tlsMinProtocolVersion</code><br/>
<em>
string
</em>
</td>
<td>
<p>TLSMinProtocolVersion defines the minimum TLS protocol version that the sidecar supports. Valid TLS protocol versions are TLS_AUTO, TLSv1_0, TLSv1_1, TLSv1_2 and TLSv1_3.</p>
</td>
</tr>
<tr>
<td>
<code>tlsMaxProtocolVersion</code><br/>
<em>
string
</em>
</td>
<td>
<p>TLSMaxProtocolVersion defines the maximum TLS protocol version that the sidecar supports. Valid TLS protocol versions are TLS_AUTO, TLSv1_0, TLSv1_1, TLSv1_2 and TLSv1_3.</p>
</td>
</tr>
<tr>
<td>
<code>cipherSuites</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>CipherSuites defines a list of ciphers that listener supports when negotiating TLS 1.0-1.2. This setting has no effect when negotiating TLS 1.3. For valid cipher names, see the latest OpenSSL ciphers manual page. E.g. <a href="https://www.openssl.org/docs/man1.1.1/apps/ciphers.html">https://www.openssl.org/docs/man1.1.1/apps/ciphers.html</a>.</p>
</td>
</tr>
<tr>
<td>
<code>ecdhCurves</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>ECDHCurves defines a list of ECDH curves that TLS connection supports. If not specified, the curves are [X25519, P-256] for non-FIPS build and P-256 for builds using BoringSSL FIPS.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.TracingSpec">TracingSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ObservabilitySpec">ObservabilitySpec</a>)
</p>
<div>
<p>TracingSpec is the type to represent OSM&rsquo;s tracing configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>enable</code><br/>
<em>
bool
</em>
</td>
<td>
<p>Enable defines a boolean indicating if the sidecars are enabled for tracing.</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
int16
</em>
</td>
<td>
<p>Port defines the tracing collector&rsquo;s port.</p>
</td>
</tr>
<tr>
<td>
<code>address</code><br/>
<em>
string
</em>
</td>
<td>
<p>Address defines the tracing collectio&rsquo;s hostname.</p>
</td>
</tr>
<tr>
<td>
<code>endpoint</code><br/>
<em>
string
</em>
</td>
<td>
<p>Endpoint defines the API endpoint for tracing requests sent to the collector.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.TrafficSpec">TrafficSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<div>
<p>TrafficSpec is the type used to represent OSM&rsquo;s traffic management configuration.</p>
</div>
<table>
<thead>
<tr>
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>enableEgress</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnableEgress defines a boolean indicating if mesh-wide Egress is enabled.</p>
</td>
</tr>
<tr>
<td>
<code>outboundIPRangeExclusionList</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>OutboundIPRangeExclusionList defines a global list of IP address ranges to exclude from outbound traffic interception by the sidecar proxy.</p>
</td>
</tr>
<tr>
<td>
<code>outboundIPRangeInclusionList</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>OutboundIPRangeInclusionList defines a global list of IP address ranges to include for outbound traffic interception by the sidecar proxy.
IP addresses outside this range will be excluded from outbound traffic interception by the sidecar proxy.</p>
</td>
</tr>
<tr>
<td>
<code>outboundPortExclusionList</code><br/>
<em>
[]int
</em>
</td>
<td>
<p>OutboundPortExclusionList defines a global list of ports to exclude from outbound traffic interception by the sidecar proxy.</p>
</td>
</tr>
<tr>
<td>
<code>inboundPortExclusionList</code><br/>
<em>
[]int
</em>
</td>
<td>
<p>InboundPortExclusionList defines a global list of ports to exclude from inbound traffic interception by the sidecar proxy.</p>
</td>
</tr>
<tr>
<td>
<code>enablePermissiveTrafficPolicyMode</code><br/>
<em>
bool
</em>
</td>
<td>
<p>EnablePermissiveTrafficPolicyMode defines a boolean indicating if permissive traffic policy mode is enabled mesh-wide.</p>
</td>
</tr>
<tr>
<td>
<code>inboundExternalAuthorization</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ExternalAuthzSpec">
ExternalAuthzSpec
</a>
</em>
</td>
<td>
<p>InboundExternalAuthorization defines a ruleset that, if enabled, will configure a remote external authorization endpoint
for all inbound and ingress traffic in the mesh.</p>
</td>
</tr>
</tbody>
</table>
<hr/>
<p><em>
Generated with <code>gen-crd-api-reference-docs</code>
on git commit <code>2820d82e</code>.
</em></p>
