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
<p>
<p>Package v1alpha2 is the v1alpha2 version of the API.</p>
</p>
Resource Types:
<ul></ul>
<h3 id="config.openservicemesh.io/v1alpha2.CertManagerProviderSpec">CertManagerProviderSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ProviderSpec">ProviderSpec</a>)
</p>
<p>
<p>CertManagerProviderSpec defines the configuration of the cert-manager provider</p>
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
<code>issuerName</code><br/>
<em>
string
</em>
</td>
<td>
<p>IssuerName specifies the name of the Issuer resource</p>
</td>
</tr>
<tr>
<td>
<code>issuerKind</code><br/>
<em>
string
</em>
</td>
<td>
<p>IssuerKind specifies the kind of Issuer</p>
</td>
</tr>
<tr>
<td>
<code>issuerGroup</code><br/>
<em>
string
</em>
</td>
<td>
<p>IssuerGroup specifies the group the Issuer belongs to</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.CertificateSpec">CertificateSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<p>
<p>CertificateSpec is the type to reperesent OSM&rsquo;s certificate management configuration.</p>
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
<h3 id="config.openservicemesh.io/v1alpha2.ExtensionService">ExtensionService
</h3>
<p>
<p>ExtensionService defines the configuration of the external service
that an OSM managed mesh integrates with.</p>
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
<p>Object&rsquo;s metadata.</p>
Refer to the Kubernetes API documentation for the fields of the
<code>metadata</code> field.
</td>
</tr>
<tr>
<td>
<code>spec</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ExtensionServiceSpec">
ExtensionServiceSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec defines the specification of the extension service.</p>
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
<p>Host defines the hostname of the extension service.</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
uint32
</em>
</td>
<td>
<p>Port defines the port number of the extension service.</p>
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
<p>Protocol defines the protocol of the extension service.</p>
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
<p>ConnectTimeout defines the timeout for connecting to the extension service.</p>
</td>
</tr>
</table>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ExtensionServiceSpec">ExtensionServiceSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ExtensionService">ExtensionService</a>)
</p>
<p>
<p>ExtensionServiceSpec defines the specification of the extension service.</p>
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
<p>Host defines the hostname of the extension service.</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
uint32
</em>
</td>
<td>
<p>Port defines the port number of the extension service.</p>
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
<p>Protocol defines the protocol of the extension service.</p>
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
<p>ConnectTimeout defines the timeout for connecting to the extension service.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ExternalAuthzSpec">ExternalAuthzSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.TrafficSpec">TrafficSpec</a>)
</p>
<p>
<p>ExternalAuthzSpec is a type to represent external authorization configuration.</p>
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
<p>
<p>FeatureFlags is a type to represent OSM&rsquo;s feature flags.</p>
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
<p>EnableEgressPolicy defines if OSM&rsquo;s EgressPolicy API is enabled.
DEPRECATED, do not use.
Disable mesh-wide global egress by setting &lsquo;spec.traffic.enableEgress&rsquo;
to &lsquo;false&rsquo; to implicitly enable the usage of EgressPolicy API.</p>
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
<p>
<p>IngressGatewayCertSpec is the type to represent the certificate specification for an ingress gateway.</p>
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
<h3 id="config.openservicemesh.io/v1alpha2.LocalProxyMode">LocalProxyMode
(<code>string</code> alias)</p></h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.SidecarSpec">SidecarSpec</a>)
</p>
<p>
<p>LocalProxyMode is a type alias representing the way the envoy sidecar proxies to the main application</p>
</p>
<table>
<thead>
<tr>
<th>Value</th>
<th>Description</th>
</tr>
</thead>
<tbody><tr><td><p>&#34;Localhost&#34;</p></td>
<td><p>LocalProxyModeLocalhost indicates the the sidecar should communicate with the main application over localhost</p>
</td>
</tr><tr><td><p>&#34;PodIP&#34;</p></td>
<td><p>LocalProxyModePodIP indicates that the sidecar should communicate with the main application via the pod ip</p>
</td>
</tr></tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshConfig">MeshConfig
</h3>
<p>
<p>MeshConfig is the type used to represent the mesh configuration.</p>
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
<p>
<p>MeshConfigSpec is the spec for OSM&rsquo;s configuration.</p>
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
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificate">MeshRootCertificate
</h3>
<p>
<p>MeshRootCertificate defines the configuration for certificate issuing
by the mesh control plane</p>
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
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateSpec">
MeshRootCertificateSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Spec is the MeshRootCertificate config specification</p>
<br/>
<br/>
<table>
<tr>
<td>
<code>provider</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ProviderSpec">
ProviderSpec
</a>
</em>
</td>
<td>
<p>Provider specifies the mesh certificate provider</p>
</td>
</tr>
<tr>
<td>
<code>trustDomain</code><br/>
<em>
string
</em>
</td>
<td>
<p>TrustDomain is the trust domain to use as a suffix in Common Names for new certificates.</p>
</td>
</tr>
<tr>
<td>
<code>intent</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateIntent">
MeshRootCertificateIntent
</a>
</em>
</td>
<td>
<p>Intent of the MeshRootCertificate resource</p>
</td>
</tr>
<tr>
<td>
<code>spiffeEnabled</code><br/>
<em>
bool
</em>
</td>
<td>
<p>SpiffeEnabled will add a SPIFFE ID to the certificates, creating a SPIFFE compatible x509 SVID document
To use SPIFFE ID for validation and routing, &lsquo;enableSPIFFE&rsquo; must be true in the MeshConfig after the MeshRootCertificate is made &lsquo;active&rsquo;</p>
</td>
</tr>
</table>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateStatus">
MeshRootCertificateStatus
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Status of the MeshRootCertificate resource</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">MeshRootCertificateComponentStatus
(<code>string</code> alias)</p></h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatuses">MeshRootCertificateComponentStatuses</a>)
</p>
<p>
<p>MeshRootCertificateComponentStatus specifies the status of the certificate component,
can be (<code>Issuing</code>, <code>Validating</code>, <code>Unknown</code>).</p>
</p>
<table>
<thead>
<tr>
<th>Value</th>
<th>Description</th>
</tr>
</thead>
<tbody><tr><td><p>&#34;issuing&#34;</p></td>
<td><p>Issuing means that the root cert described by this MRC is now issuing certs for this component of OSM.</p>
</td>
</tr><tr><td><p>&#34;unknown&#34;</p></td>
<td><p>UnknownComponentStatus means that the use of the root cert described by this MRC is in an unknown state for this
component.</p>
</td>
</tr><tr><td><p>&#34;unused&#34;</p></td>
<td><p>Unused means that the root cert described by this MRC is unused.</p>
</td>
</tr><tr><td><p>&#34;validating&#34;</p></td>
<td><p>Validating means that the root cert&rsquo;s cert chain, described by this MRC is now part of the CABundle used to
validate requests for this component..</p>
</td>
</tr></tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatuses">MeshRootCertificateComponentStatuses
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateStatus">MeshRootCertificateStatus</a>)
</p>
<p>
<p>MeshRootCertificateComponentStatuses is the set of statuses for each certificate component in the cluster.</p>
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
<code>validatingWebhook</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
<tr>
<td>
<code>mutatingWebhook</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
<tr>
<td>
<code>xdsControlPlane</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
<tr>
<td>
<code>sidecar</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
<tr>
<td>
<code>bootstrap</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
<tr>
<td>
<code>gateway</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatus">
MeshRootCertificateComponentStatus
</a>
</em>
</td>
<td>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateCondition">MeshRootCertificateCondition
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateStatus">MeshRootCertificateStatus</a>)
</p>
<p>
<p>MeshRootCertificateCondition defines the condition of the MeshRootCertificate resource.</p>
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
<code>type</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateConditionType">
MeshRootCertificateConditionType
</a>
</em>
</td>
<td>
<p>Type of the condition,
one of (<code>Ready</code>, <code>Accepted</code>, <code>IssuingRollout</code>, <code>ValidatingRollout</code>, <code>IssuingRollback</code>, <code>ValidatingRollback</code>).</p>
</td>
</tr>
<tr>
<td>
<code>status</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateConditionStatus">
MeshRootCertificateConditionStatus
</a>
</em>
</td>
<td>
<p>Status of the condition, one of (<code>True</code>, <code>False</code>, <code>Unknown</code>).</p>
</td>
</tr>
<tr>
<td>
<code>lastTransitionTime</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#time-v1-meta">
Kubernetes meta/v1.Time
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>LastTransitionTime is the timestamp corresponding to the last status
change of this condition.</p>
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
<p>Reason is a brief machine readable explanation for the condition&rsquo;s last
transition (should be in camelCase).</p>
</td>
</tr>
<tr>
<td>
<code>message</code><br/>
<em>
string
</em>
</td>
<td>
<em>(Optional)</em>
<p>Message is a human readable description of the details of the last
transition, complementing reason.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateConditionStatus">MeshRootCertificateConditionStatus
(<code>string</code> alias)</p></h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateCondition">MeshRootCertificateCondition</a>)
</p>
<p>
<p>MeshRootCertificateConditionStatus specifies the status of the MeshRootCertificate condition,
one of (<code>True</code>, <code>False</code>, <code>Unknown</code>).</p>
</p>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateConditionType">MeshRootCertificateConditionType
(<code>string</code> alias)</p></h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateCondition">MeshRootCertificateCondition</a>)
</p>
<p>
<p>MeshRootCertificateConditionType specifies the type of the condition,
one of (<code>Ready</code>, <code>Accepted</code>, <code>IssuingRollout</code>, <code>ValidatingRollout</code>, <code>IssuingRollback</code>, <code>ValidatingRollback</code>).</p>
</p>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateIntent">MeshRootCertificateIntent
(<code>string</code> alias)</p></h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateSpec">MeshRootCertificateSpec</a>)
</p>
<p>
<p>MeshRootCertificateIntent specifies the intent of the MeshRootCertificate
can be (Active, Passive).</p>
</p>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateSpec">MeshRootCertificateSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificate">MeshRootCertificate</a>)
</p>
<p>
<p>MeshRootCertificateSpec defines the mesh root certificate specification</p>
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
<code>provider</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.ProviderSpec">
ProviderSpec
</a>
</em>
</td>
<td>
<p>Provider specifies the mesh certificate provider</p>
</td>
</tr>
<tr>
<td>
<code>trustDomain</code><br/>
<em>
string
</em>
</td>
<td>
<p>TrustDomain is the trust domain to use as a suffix in Common Names for new certificates.</p>
</td>
</tr>
<tr>
<td>
<code>intent</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateIntent">
MeshRootCertificateIntent
</a>
</em>
</td>
<td>
<p>Intent of the MeshRootCertificate resource</p>
</td>
</tr>
<tr>
<td>
<code>spiffeEnabled</code><br/>
<em>
bool
</em>
</td>
<td>
<p>SpiffeEnabled will add a SPIFFE ID to the certificates, creating a SPIFFE compatible x509 SVID document
To use SPIFFE ID for validation and routing, &lsquo;enableSPIFFE&rsquo; must be true in the MeshConfig after the MeshRootCertificate is made &lsquo;active&rsquo;</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.MeshRootCertificateStatus">MeshRootCertificateStatus
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificate">MeshRootCertificate</a>)
</p>
<p>
<p>MeshRootCertificateStatus defines the status of the MeshRootCertificate resource.</p>
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
<code>state</code><br/>
<em>
string
</em>
</td>
<td>
<p>State specifies the state of the certificate provider.
All states are specified in constants.go</p>
</td>
</tr>
<tr>
<td>
<code>transitionAfter</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#time-v1-meta">
Kubernetes meta/v1.Time
</a>
</em>
</td>
<td>
<p>If present, this MRC can transition to the next state in the state machine after this timestamp.</p>
</td>
</tr>
<tr>
<td>
<code>componentStatuses</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateComponentStatuses">
MeshRootCertificateComponentStatuses
</a>
</em>
</td>
<td>
<p>Set of statuses for each certificate component in the cluster (e.g. webhooks, bootstrap, etc.)
NOTE: There is a caveat that since these components belong to horizontally scalable pods, it is possible that not
all of these components will be ready. That is, one controller might mark the ADS server as ready, while all other
controllers have yet to rotate their controller cert.</p>
</td>
</tr>
<tr>
<td>
<code>conditions</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateCondition">
[]MeshRootCertificateCondition
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>List of status conditions to indicate the status of a MeshRootCertificate.
Known condition types are <code>Ready</code> and <code>InvalidRequest</code>.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.ObservabilitySpec">ObservabilitySpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<p>
<p>ObservabilitySpec is the type to represent OSM&rsquo;s observability configurations.</p>
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
<h3 id="config.openservicemesh.io/v1alpha2.ProviderSpec">ProviderSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshRootCertificateSpec">MeshRootCertificateSpec</a>)
</p>
<p>
<p>ProviderSpec defines the certificate provider used by the mesh control plane</p>
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
<code>certManager</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.CertManagerProviderSpec">
CertManagerProviderSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>CertManager specifies the cert-manager provider configuration</p>
</td>
</tr>
<tr>
<td>
<code>vault</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.VaultProviderSpec">
VaultProviderSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Vault specifies the vault provider configuration</p>
</td>
</tr>
<tr>
<td>
<code>tresor</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.TresorProviderSpec">
TresorProviderSpec
</a>
</em>
</td>
<td>
<em>(Optional)</em>
<p>Tresor specifies the Tresor provider configuration</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.SecretKeyReferenceSpec">SecretKeyReferenceSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.VaultTokenSpec">VaultTokenSpec</a>)
</p>
<p>
<p>SecretKeyReferenceSpec defines the configuration of the secret reference</p>
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
<p>Name specifies the name of the secret in which the Vault token is stored</p>
</td>
</tr>
<tr>
<td>
<code>key</code><br/>
<em>
string
</em>
</td>
<td>
<p>Key specifies the key whose value is the Vault token</p>
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
<p>Namespace specifies the namespace of the secret in which the Vault token is stored</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.SidecarSpec">SidecarSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.MeshConfigSpec">MeshConfigSpec</a>)
</p>
<p>
<p>SidecarSpec is the type used to represent the specifications for the proxy sidecar.</p>
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
<p>TLSMaxProtocolVersion defines the maximum TLS protocol version that the sidecar supports. Valid TLS protocol versions are TLS_AUTO, TLSv1_0 (deprecated), TLSv1_1 (deprecated), TLSv1_2 and TLSv1_3.</p>
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
<tr>
<td>
<code>localProxyMode</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.LocalProxyMode">
LocalProxyMode
</a>
</em>
</td>
<td>
<p>LocalProxyMode defines the network interface the envoy proxy will use to send traffic to the backend service application. Acceptable values are [<code>Localhost</code>, <code>PodIP</code>]. The default is <code>Localhost</code></p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.TracingSpec">TracingSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ObservabilitySpec">ObservabilitySpec</a>)
</p>
<p>
<p>TracingSpec is the type to represent OSM&rsquo;s tracing configuration.</p>
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
<p>
<p>TrafficSpec is the type used to represent OSM&rsquo;s traffic management configuration.</p>
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
<tr>
<td>
<code>networkInterfaceExclusionList</code><br/>
<em>
[]string
</em>
</td>
<td>
<p>NetworkInterfaceExclusionList defines a global list of network interface
names to exclude from inbound and outbound traffic interception by the
sidecar proxy.</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.TresorCASpec">TresorCASpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.TresorProviderSpec">TresorProviderSpec</a>)
</p>
<p>
<p>TresorCASpec defines the configuration of Tresor&rsquo;s root certificate</p>
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
<code>secretRef</code><br/>
<em>
<a href="https://v1-20.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#secretreference-v1-core">
Kubernetes core/v1.SecretReference
</a>
</em>
</td>
<td>
<p>SecretRef specifies the secret in which the root certificate is stored</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.TresorProviderSpec">TresorProviderSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ProviderSpec">ProviderSpec</a>)
</p>
<p>
<p>TresorProviderSpec defines the configuration of the Tresor provider</p>
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
<code>ca</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.TresorCASpec">
TresorCASpec
</a>
</em>
</td>
<td>
<p>CA specifies Tresor&rsquo;s ca configuration</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.VaultProviderSpec">VaultProviderSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.ProviderSpec">ProviderSpec</a>)
</p>
<p>
<p>VaultProviderSpec defines the configuration of the Vault provider</p>
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
<p>Host specifies the name of the Vault server</p>
</td>
</tr>
<tr>
<td>
<code>port</code><br/>
<em>
int
</em>
</td>
<td>
<p>Port specifies the port of the Vault server</p>
</td>
</tr>
<tr>
<td>
<code>role</code><br/>
<em>
string
</em>
</td>
<td>
<p>Role specifies the name of the role for use by mesh control plane</p>
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
<p>Protocol specifies the protocol for connections to Vault</p>
</td>
</tr>
<tr>
<td>
<code>token</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.VaultTokenSpec">
VaultTokenSpec
</a>
</em>
</td>
<td>
<p>Token specifies the configuration of the token to be used by mesh control plane
to connect to Vault</p>
</td>
</tr>
</tbody>
</table>
<h3 id="config.openservicemesh.io/v1alpha2.VaultTokenSpec">VaultTokenSpec
</h3>
<p>
(<em>Appears on:</em><a href="#config.openservicemesh.io/v1alpha2.VaultProviderSpec">VaultProviderSpec</a>)
</p>
<p>
<p>VaultTokenSpec defines the configuration of the Vault token</p>
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
<code>secretKeyRef</code><br/>
<em>
<a href="#config.openservicemesh.io/v1alpha2.SecretKeyReferenceSpec">
SecretKeyReferenceSpec
</a>
</em>
</td>
<td>
<p>SecretKeyRef specifies the secret in which the Vault token is stored</p>
</td>
</tr>
</tbody>
</table>
<hr/>
<p><em>
Generated with <code>gen-crd-api-reference-docs</code>
on git commit <code>a65cd374</code>.
</em></p>
