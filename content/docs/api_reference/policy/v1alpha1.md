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
<p>Kind defines the kind for the source in the IngressBackend policy.</p>
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
<hr/>
<p><em>
Generated with <code>gen-crd-api-reference-docs</code>
on git commit <code>3265b4e2</code>.
</em></p>
