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
<div>
<p>Package v1alpha1 is the v1alpha1 version of the API.</p>
</div>
Resource Types:
<ul></ul>
<h3 id="policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.IngressBackendSpec">IngressBackendSpec</a>)
</p>
<div>
<p>BackendSpec is the type used to represent a Backend specified in the IngressBackend policy specification.</p>
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
<div>
<p>ConnectionSettingsSpec defines the connection settings for an
upstream host.</p>
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
<div>
<p>Egress is the type used to represent an Egress traffic policy.
An Egress policy allows applications to access endpoints
external to the service mesh or cluster based on the specified
rules in the policy.</p>
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
<div>
<p>EgressSourceSpec is the type used to represent the Source in the list of Sources specified in an Egress policy specification.</p>
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
<div>
<p>EgressSpec is the type used to represent the Egress policy specification.</p>
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
<div>
<p>HTTPConnectionSettings defines the HTTP connection settings for an
upstream host.</p>
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
<h3 id="policy.openservicemesh.io/v1alpha1.IngressBackend">IngressBackend
</h3>
<div>
<p>IngressBackend is the type used to represent an Ingress backend policy.
An Ingress backend policy authorizes one or more backends to accept
ingress traffic from one or more sources.</p>
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
<div>
<p>IngressBackendSpec is the type used to represent the IngressBackend policy specification.</p>
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
<div>
<p>IngressBackendStatus is the type used to represent the status of an IngressBackend resource.</p>
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
<div>
<p>IngressSourceSpec is the type used to represent the Source in the list of Sources specified in an
IngressBackend policy specification.</p>
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
<h3 id="policy.openservicemesh.io/v1alpha1.PortSpec">PortSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec</a>, <a href="#policy.openservicemesh.io/v1alpha1.EgressSpec">EgressSpec</a>)
</p>
<div>
<p>PortSpec is the type used to represent the Port in the list of Ports specified in an Egress policy specification.</p>
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
<h3 id="policy.openservicemesh.io/v1alpha1.Retry">Retry
</h3>
<div>
<p>Retry is the type used to represent a Retry policy.
A Retry policy authorizes retries to failed attempts for outbound traffic
from one service source to one or more destination services.</p>
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
<div>
<p>RetryPolicySpec is the type used to represent the retry policy specified in the Retry policy specification.</p>
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
string
</em>
</td>
<td>
<p>PerTryTimeout defines the time allowed for a retry before it&rsquo;s considered a failed attempt.</p>
</td>
</tr>
<tr>
<td>
<code>numRetries</code><br/>
<em>
int
</em>
</td>
<td>
<p>NumRetries defines the max number of retries to attempt.</p>
</td>
</tr>
<tr>
<td>
<code>retryBackoffInterval</code><br/>
<em>
string
</em>
</td>
<td>
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
<div>
<p>RetrySpec is the type used to represent the Retry policy specification.</p>
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
<div>
<p>RetrySrcDstSpec is the type used to represent the Destination in the list of Destinations and the Source
specified in the Retry policy specification.</p>
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
<div>
<p>TCPConnectionSettings defines the TCP connection settings for an
upstream host.</p>
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
<h3 id="policy.openservicemesh.io/v1alpha1.TLSSpec">TLSSpec
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.BackendSpec">BackendSpec</a>)
</p>
<div>
<p>TLSSpec is the type used to represent the backend&rsquo;s TLS configuration.</p>
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
<div>
<p>UpstreamTrafficSetting defines the settings applicable to traffic destined
to an upstream host.</p>
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
<div>
<p>UpstreamTrafficSettingSpec defines the upstream traffic setting specification.</p>
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
</tbody>
</table>
<h3 id="policy.openservicemesh.io/v1alpha1.UpstreamTrafficSettingStatus">UpstreamTrafficSettingStatus
</h3>
<p>
(<em>Appears on:</em><a href="#policy.openservicemesh.io/v1alpha1.UpstreamTrafficSetting">UpstreamTrafficSetting</a>)
</p>
<div>
<p>UpstreamTrafficSettingStatus defines the status of an UpstreamTrafficSetting resource.</p>
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
on git commit <code>407bbedd5</code>.
</em></p>
