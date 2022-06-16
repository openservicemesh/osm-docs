---
title: "Policy v1alpha1 API Reference"
description: "Policy v1alpha1 API reference documentation."
type: docs
---

<p>Packages:</p>
<ul>
<li>
<a href="#policy.openservicemesh.io%2fv1alpha1">policy.openservicemesh.io/v1alpha1</a>
</li>
</ul>
<h2 id="policy.openservicemesh.io/v1alpha1">policy.openservicemesh.io/v1alpha1</h2>
<p>
<p>Package v1alpha1 is the v1alpha1 version of the API.</p>
</p>
Resource Types:
<ul></ul>
<h3 id="policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpec</a>)
</p>
<p>
<p>BackendSpec is the type used to represent a Backend specified in the IngressBackend policy specification.</p>
</p>
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
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name defines the name of the backend.</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
PortSpec
</a>
</em>
</td>
<td>
<p>Port defines the specification for the backend&rsquo;s port.</p>
</td>
</tr>
<tr>
<td>
<code>tls</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.TLSSpec">
TLSSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>TLS defines the specification for the backend&rsquo;s TLS configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">ConnectionSettingsSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">UpstreamTrafficSettingSpec</a>)
</p>
<p>
<p>ConnectionSettingsSpec defines the connection settings for an
upstream host.</p>
</p>
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
<code>tcp</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.TCPConnectionSettings">
TCPConnectionSettings
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>TCP specifies the TCP level connection settings.
Applies to both TCP and HTTP connections.</p>
</td>
</tr>
<tr>
<td>
<code>http</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPConnectionSettings">
HTTPConnectionSettings
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>HTTP specifies the HTTP level connection settings.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.Egress">Egress
</h3>
<p>
<p>Egress is the type used to represent an Egress traffic policy.
An Egress policy allows applications to access endpoints
external to the service mesh or cluster based on the specified
rules in the policy.</p>
</p>
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
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">
EgressSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the Egress policy specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSourceSpec">
[]EgressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources defines the list of sources the Egress policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>hosts</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Hosts defines the list of external hosts the Egress policy will allow
access to.</p>
<ul>
<li><p>For HTTP traffic, the HTTP Host/Authority header is matched against the
list of Hosts specified.</p></li>
<li><p>For HTTPS traffic, the Server Name Indication (SNI) indicated by the client
in the TLS handshake is matched against the list of Hosts specified.</p></li>
<li><p>For non-HTTP(s) based protocols, the Hosts field is ignored.</p></li>
</ul>
</td>
</tr>
<tr>
<td>
<code>ipAddresses</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>(Optional)</em>
<p>IPAddresses defines the list of external IP address ranges the Egress policy
applies to. The destination IP address of the traffic is matched against the
list of IPAddresses specified as a CIDR range.</p>
</td>
</tr>
<tr>
<td>
<code>ports</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>Ports defines the list of ports the Egress policy is applies to.
The destination port of the traffic is matched against the list of Ports specified.</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Matches defines the list of object references the Egress policy should match on.</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.EgressSourceSpec">EgressSourceSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpec</a>)
</p>
<p>
<p>EgressSourceSpec is the type used to represent the Source in the list of Sources specified in an Egress policy specification.</p>
</p>
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
<code>kind</code><br/>
<em>
string
</em>
</td>
<td>
<p>Kind defines the kind for the source in the Egress policy, ex. ServiceAccount.</p>
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
<p>Name defines the name of the source for the given Kind.</p>
</td>
</tr>
<tr>
<td>
<code>namespace</code><br/>
<em>
string
</em>
</td>
<td>
<p>Namespace defines the namespace for the given source.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.Egress">Egress</a>)
</p>
<p>
<p>EgressSpec is the type used to represent the Egress policy specification.</p>
</p>
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
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.EgressSourceSpec">
[]EgressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources defines the list of sources the Egress policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>hosts</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Hosts defines the list of external hosts the Egress policy will allow
access to.</p>
<ul>
<li><p>For HTTP traffic, the HTTP Host/Authority header is matched against the
list of Hosts specified.</p></li>
<li><p>For HTTPS traffic, the Server Name Indication (SNI) indicated by the client
in the TLS handshake is matched against the list of Hosts specified.</p></li>
<li><p>For non-HTTP(s) based protocols, the Hosts field is ignored.</p></li>
</ul>
</td>
</tr>
<tr>
<td>
<code>ipAddresses</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>(Optional)</em>
<p>IPAddresses defines the list of external IP address ranges the Egress policy
applies to. The destination IP address of the traffic is matched against the
list of IPAddresses specified as a CIDR range.</p>
</td>
</tr>
<tr>
<td>
<code>ports</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.PortSpec">
[]PortSpec
</a>
</em>
</td>
<td>
<p>Ports defines the list of ports the Egress policy is applies to.
The destination port of the traffic is matched against the list of Ports specified.</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Matches defines the list of object references the Egress policy should match on.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPConnectionSettings">HTTPConnectionSettings
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">ConnectionSettingsSpec</a>)
</p>
<p>
<p>HTTPConnectionSettings defines the HTTP connection settings for an
upstream host.</p>
</p>
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
<code>maxRequests</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>MaxRequests specifies the maximum number of parallel requests
allowed to the upstream host.
Defaults to 4294967295 (2^32 - 1) if not specified.</p>
</td>
</tr>
<tr>
<td>
<code>maxRequestsPerConnection</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>MaxRequestsPerConnection specifies the maximum number of requests
per connection allowed to the upstream host.
Defaults to unlimited if not specified.</p>
</td>
</tr>
<tr>
<td>
<code>maxPendingRequests</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>MaxPendingRequests specifies the maximum number of pending HTTP
requests allowed to the upstream host. For HTTP/2 connections,
if <code>maxRequestsPerConnection</code> is not configured, all requests will
be multiplexed over the same connection so this circuit breaker
will only be hit when no connection is already established.
Defaults to 4294967295 (2^32 - 1) if not specified.</p>
</td>
</tr>
<tr>
<td>
<code>maxRetries</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>MaxRetries specifies the maximum number of parallel retries
allowed to the upstream host.
Defaults to 4294967295 (2^32 - 1) if not specified.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPHeaderValue">HTTPHeaderValue
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.HTTPLocalRateLimitSpec">HTTPLocalRateLimitSpec</a>)
</p>
<p>
<p>HTTPHeaderValue defines an HTTP header name/value pair</p>
</p>
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
<code>name</code><br/>
<em>
string
</em>
</td>
<td>
<p>Name defines the name of the HTTP header.</p>
</td>
</tr>
<tr>
<td>
<code>value</code><br/>
<em>
string
</em>
</td>
<td>
<p>Value defines the value of the header corresponding to the name key.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPLocalRateLimitSpec">HTTPLocalRateLimitSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.HTTPPerRouteRateLimitSpec">HTTPPerRouteRateLimitSpec</a>, <a href="#policy.openservicemesh.io/v1alpha1.LocalRateLimitSpec">LocalRateLimitSpec</a>)
</p>
<p>
<p>HTTPLocalRateLimitSpec defines the local rate limiting specification
for the upstream host at the HTTP level.</p>
</p>
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
<code>requests</code><br/>
<em>
uint32
</em>
</td>
<td>
<p>Requests defines the number of requests allowed
per unit of time before rate limiting occurs.</p>
</td>
</tr>
<tr>
<td>
<code>unit</code><br/>
<em>
string
</em>
</td>
<td>
<p>Unit defines the period of time within which requests
over the limit will be rate limited.
Valid values are &ldquo;second&rdquo;, &ldquo;minute&rdquo; and &ldquo;hour&rdquo;.</p>
</td>
</tr>
<tr>
<td>
<code>burst</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>Burst defines the number of requests above the baseline
rate that are allowed in a short period of time.</p>
</td>
</tr>
<tr>
<td>
<code>responseStatusCode</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>ResponseStatusCode defines the HTTP status code to use for responses
to rate limited requests. Code must be in the 400-599 (inclusive)
error range. If not specified, a default of 429 (Too Many Requests) is used.</p>
</td>
</tr>
<tr>
<td>
<code>responseHeadersToAdd</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPHeaderValue">
[]HTTPHeaderValue
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>ResponseHeadersToAdd defines the list of HTTP headers that should be
added to each response for requests that have been rate limited.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPPerRouteRateLimitSpec">HTTPPerRouteRateLimitSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.HTTPRouteSpec">HTTPRouteSpec</a>)
</p>
<p>
<p>HTTPPerRouteRateLimitSpec defines the rate limiting specification
per HTTP route.</p>
</p>
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
<code>local</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPLocalRateLimitSpec">
HTTPLocalRateLimitSpec
</a>
</em>
</td>
<td>
<p>Local defines the local rate limiting specification
applied per HTTP route.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.HTTPRouteSpec">HTTPRouteSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">UpstreamTrafficSettingSpec</a>)
</p>
<p>
<p>HTTPRouteSpec defines the settings correspondng to an HTTP route</p>
</p>
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
<code>path</code><br/>
<em>
string
</em>
</td>
<td>
<p>Path defines the HTTP path.</p>
</td>
</tr>
<tr>
<td>
<code>rateLimit</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPPerRouteRateLimitSpec">
HTTPPerRouteRateLimitSpec
</a>
</em>
</td>
<td>
<p>RateLimit defines the HTTP rate limiting specification for
the specified HTTP route.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackend
</h3>
<p>
<p>IngressBackend is the type used to represent an Ingress backend policy.
An Ingress backend policy authorizes one or more backends to accept
ingress traffic from one or more sources.</p>
</p>
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
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">
IngressBackendSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the Ingress backend policy specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>backends</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">
[]BackendSpec
</a>
</em>
</td>
<td>
<p>Backends defines the list of backends the IngressBackend policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressSourceSpec">
[]IngressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources defines the list of sources the IngressBackend policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Matches defines the list of object references the IngressBackend policy should match on.</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressBackendStatus">
IngressBackendStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status is the status of the IngressBackend configuration.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackend</a>)
</p>
<p>
<p>IngressBackendSpec is the type used to represent the IngressBackend policy specification.</p>
</p>
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
<code>backends</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">
[]BackendSpec
</a>
</em>
</td>
<td>
<p>Backends defines the list of backends the IngressBackend policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>sources</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.IngressSourceSpec">
[]IngressSourceSpec
</a>
</em>
</td>
<td>
<p>Sources defines the list of sources the IngressBackend policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>matches</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#typedlocalobjectreference-v1-core">
[]Kubernetes core/v1.TypedLocalObjectReference
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Matches defines the list of object references the IngressBackend policy should match on.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackendStatus">IngressBackendStatus
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackend</a>)
</p>
<p>
<p>IngressBackendStatus is the type used to represent the status of an IngressBackend resource.</p>
</p>
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
<code>currentStatus</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>CurrentStatus defines the current status of an IngressBackend resource.</p>
</td>
</tr>
<tr>
<td>
<code>reason</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Reason defines the reason for the current status of an IngressBackend resource.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.IngressSourceSpec">IngressSourceSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpec</a>)
</p>
<p>
<p>IngressSourceSpec is the type used to represent the Source in the list of Sources specified in an
IngressBackend policy specification.</p>
</p>
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
<code>kind</code><br/>
<em>
string
</em>
</td>
<td>
<p>Kind defines the kind for the source in the IngressBackend policy.
Must be one of: Service, AuthenticatedPrincipal, IPRange</p>
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
<p>Name defines the name of the source for the given Kind.</p>
</td>
</tr>
<tr>
<td>
<code>namespace</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Namespace defines the namespace for the given source.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.LocalRateLimitSpec">LocalRateLimitSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.RateLimitSpec">RateLimitSpec</a>)
</p>
<p>
<p>LocalRateLimitSpec defines the local rate limiting specification
for the upstream host.</p>
</p>
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
<code>tcp</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.TCPLocalRateLimitSpec">
TCPLocalRateLimitSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>TCP defines the local rate limiting specification at the network
level. This is a token bucket rate limiter where each connection
consumes a single token. If the token is available, the connection
will be allowed. If no tokens are available, the connection will be
immediately closed.</p>
</td>
</tr>
<tr>
<td>
<code>http</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPLocalRateLimitSpec">
HTTPLocalRateLimitSpec
</a>
</em>
</td>
<td>
<p>HTTP defines the local rate limiting specification for HTTP traffic.
This is a token bucket rate limiter where each request consumes
a single token. If the token is available, the request will be
allowed. If no tokens are available, the request will receive the
configured rate limit status.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.PortSpec">PortSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec</a>, <a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpec</a>)
</p>
<p>
<p>PortSpec is the type used to represent the Port in the list of Ports specified in an Egress policy specification.</p>
</p>
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
<code>number</code><br/>
<em>
int
</em>
</td>
<td>
<p>Number defines the port number.</p>
</td>
</tr>
<tr>
<td>
<code>protocol</code><br/>
<em>
string
</em>
</td>
<td>
<p>Protocol defines the protocol served by the port.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RateLimitSpec">RateLimitSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">UpstreamTrafficSettingSpec</a>)
</p>
<p>
<p>RateLimitSpec defines the rate limiting specification for
the upstream host.</p>
</p>
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
<code>local</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.LocalRateLimitSpec">
LocalRateLimitSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Local specified the local rate limiting specification
for the upstream host.
Local rate limiting is enforced directly by the upstream
host without any involvement of a global rate limiting service.
This is applied as a token bucket rate limiter.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.Retry">Retry
</h3>
<p>
<p>Retry is the type used to represent a Retry policy.
A Retry policy authorizes retries to failed attempts for outbound traffic
from one service source to one or more destination services.</p>
</p>
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
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">
RetrySpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the Retry policy specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>source</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Source defines the source the Retry policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>destinations</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
[]RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Destinations defines the list of destinations the Retry policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>retryPolicy</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetryPolicySpec">
RetryPolicySpec
</a>
</em>
</td>
<td>
<p>RetryPolicy defines the retry policy the Retry policy applies.</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetryPolicySpec">RetryPolicySpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">RetrySpec</a>)
</p>
<p>
<p>RetryPolicySpec is the type used to represent the retry policy specified in the Retry policy specification.</p>
</p>
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
<code>retryOn</code><br/>
<em>
string
</em>
</td>
<td>
<p>RetryOn defines the policies to retry on, delimited by comma.</p>
</td>
</tr>
<tr>
<td>
<code>perTryTimeout</code><br/>
<em>
<a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Duration">
Kubernetes meta/v1.Duration
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>PerTryTimeout defines the time allowed for a retry before it&rsquo;s considered a failed attempt.</p>
</td>
</tr>
<tr>
<td>
<code>numRetries</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>NumRetries defines the max number of retries to attempt.</p>
</td>
</tr>
<tr>
<td>
<code>retryBackoffBaseInterval</code><br/>
<em>
<a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Duration">
Kubernetes meta/v1.Duration
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>RetryBackoffBaseInterval defines the base interval for exponential retry backoff.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetrySpec">RetrySpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.Retry">Retry</a>)
</p>
<p>
<p>RetrySpec is the type used to represent the Retry policy specification.</p>
</p>
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
<code>source</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Source defines the source the Retry policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>destinations</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">
[]RetrySrcDstSpec
</a>
</em>
</td>
<td>
<p>Destinations defines the list of destinations the Retry policy applies to.</p>
</td>
</tr>
<tr>
<td>
<code>retryPolicy</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RetryPolicySpec">
RetryPolicySpec
</a>
</em>
</td>
<td>
<p>RetryPolicy defines the retry policy the Retry policy applies.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.RetrySrcDstSpec">RetrySrcDstSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.RetrySpec">RetrySpec</a>)
</p>
<p>
<p>RetrySrcDstSpec is the type used to represent the Destination in the list of Destinations and the Source
specified in the Retry policy specification.</p>
</p>
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
<code>kind</code><br/>
<em>
string
</em>
</td>
<td>
<p>Kind defines the kind for the Src/Dst in the Retry policy.</p>
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
<p>Name defines the name of the Src/Dst for the given Kind.</p>
</td>
</tr>
<tr>
<td>
<code>namespace</code><br/>
<em>
string
</em>
</td>
<td>
<p>Namespace defines the namespace for the given Src/Dst.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.TCPConnectionSettings">TCPConnectionSettings
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">ConnectionSettingsSpec</a>)
</p>
<p>
<p>TCPConnectionSettings defines the TCP connection settings for an
upstream host.</p>
</p>
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
<code>maxConnections</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>MaxConnections specifies the maximum number of TCP connections
allowed to the upstream host.
Defaults to 4294967295 (2^32 - 1) if not specified.</p>
</td>
</tr>
<tr>
<td>
<code>connectTimeout</code><br/>
<em>
<a href="https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#Duration">
Kubernetes meta/v1.Duration
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>ConnectTimeout specifies the TCP connection timeout.
Defaults to 5s if not specified.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.TCPLocalRateLimitSpec">TCPLocalRateLimitSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.LocalRateLimitSpec">LocalRateLimitSpec</a>)
</p>
<p>
<p>TCPLocalRateLimitSpec defines the local rate limiting specification
for the upstream host at the TCP level.</p>
</p>
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
<code>connections</code><br/>
<em>
uint32
</em>
</td>
<td>
<p>Connections defines the number of connections allowed
per unit of time before rate limiting occurs.</p>
</td>
</tr>
<tr>
<td>
<code>unit</code><br/>
<em>
string
</em>
</td>
<td>
<p>Unit defines the period of time within which connections
over the limit will be rate limited.
Valid values are &ldquo;second&rdquo;, &ldquo;minute&rdquo; and &ldquo;hour&rdquo;.</p>
</td>
</tr>
<tr>
<td>
<code>burst</code><br/>
<em>
uint32
</em>
</td>
<td>
<em>(Optional)</em>
<p>Burst defines the number of connections above the baseline
rate that are allowed in a short period of time.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.TLSSpec">TLSSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec</a>)
</p>
<p>
<p>TLSSpec is the type used to represent the backend&rsquo;s TLS configuration.</p>
</p>
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
<code>skipClientCertValidation</code><br/>
<em>
bool
</em>
</td>
<td>
<p>SkipClientCertValidation defines whether the backend should skip validating the
certificate presented by the client.</p>
</td>
</tr>
<tr>
<td>
<code>sniHosts</code><br/>
<em>
[]string
</em>
</td>
<td>
<em>(Optional)</em>
<p>SNIHosts defines the SNI hostnames that the backend allows the client to connect to.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSetting
</h3>
<p>
<p>UpstreamTrafficSetting defines the settings applicable to traffic destined
to an upstream host.</p>
</p>
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
<p>Object&rsquo;s metadata</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">
UpstreamTrafficSettingSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the UpstreamTrafficSetting policy specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>host</code><br/>
<em>
string
</em>
</td>
<td>
<p>Host the upstream traffic is directed to.
Must either be an FQDN corresponding to the upstream service
or the name of the upstream service. If only the service name
is specified, the FQDN is derived from the service name and
the namespace of the UpstreamTrafficSetting rule.</p>
</td>
</tr>
<tr>
<td>
<code>connectionSettings</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">
ConnectionSettingsSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>ConnectionSettings specifies the connection settings for traffic
directed to the upstream host.</p>
</td>
</tr>
<tr>
<td>
<code>rateLimit</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RateLimitSpec">
RateLimitSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>RateLimit specifies the rate limit settings for the traffic
directed to the upstream host.
If HTTP rate limiting is specified, the rate limiting is applied
at the VirtualHost level applicable to all routes within the
VirtualHost.</p>
</td>
</tr>
<tr>
<td>
<code>httpRoutes</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPRouteSpec">
[]HTTPRouteSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>HTTPRoutes defines the list of HTTP routes settings
for the upstream host. Settings are applied at a per
route level.</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingStatus">
UpstreamTrafficSettingStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status is the status of the UpstreamTrafficSetting resource.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingSpec">UpstreamTrafficSettingSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSetting</a>)
</p>
<p>
<p>UpstreamTrafficSettingSpec defines the upstream traffic setting specification.</p>
</p>
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
<code>host</code><br/>
<em>
string
</em>
</td>
<td>
<p>Host the upstream traffic is directed to.
Must either be an FQDN corresponding to the upstream service
or the name of the upstream service. If only the service name
is specified, the FQDN is derived from the service name and
the namespace of the UpstreamTrafficSetting rule.</p>
</td>
</tr>
<tr>
<td>
<code>connectionSettings</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.ConnectionSettingsSpec">
ConnectionSettingsSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>ConnectionSettings specifies the connection settings for traffic
directed to the upstream host.</p>
</td>
</tr>
<tr>
<td>
<code>rateLimit</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.RateLimitSpec">
RateLimitSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>RateLimit specifies the rate limit settings for the traffic
directed to the upstream host.
If HTTP rate limiting is specified, the rate limiting is applied
at the VirtualHost level applicable to all routes within the
VirtualHost.</p>
</td>
</tr>
<tr>
<td>
<code>httpRoutes</code><br/>
<em>
<a href="#policy.openservicemesh.io/v1alpha1.HTTPRouteSpec">
[]HTTPRouteSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>HTTPRoutes defines the list of HTTP routes settings
for the upstream host. Settings are applied at a per
route level.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingStatus">UpstreamTrafficSettingStatus
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSetting</a>)
</p>
<p>
<p>UpstreamTrafficSettingStatus defines the status of an UpstreamTrafficSetting resource.</p>
</p>
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
<code>currentStatus</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>CurrentStatus defines the current status of an UpstreamTrafficSetting resource.</p>
</td>
</tr>
<tr>
<td>
<code>reason</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Reason defines the reason for the current status of an UpstreamTrafficSetting resource.</p>
</td>
</tr>
</tbody>
</table>
<hr/>
<p><em>
Generated with <code>gen-crd-api-reference-docs</code>
on git commit <code>a5b37165</code>.
</em></p>
