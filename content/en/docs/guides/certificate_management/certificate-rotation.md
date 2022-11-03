---
title: "Root Certificate Rotation"
description: "Rotating the root certificate in OSM"
type: docs
weight: 10
---

# Root Certificate Rotation

The Mesh's Root certificate is a long-lived certificate that is used by OSM to issue leaf certificates to services in the mesh thereby enabling mTLS. This document focuses on the steps to rotate a certificate, to learn more about OSM's certificates see [Certificate Provider Options](certificates.md).  

The certificate can be rotated either through a [manual process](#manual-root-certificate-rotation) or with OSM v1.3 we have introduced a preview feature to help [automate rotation](#certificate-rotation-with-meshrootcertificate) and minimize downtime using a new API called the `MeshRootCertificate`.  

### Manual Root Certificate Rotation

#### Tresor

>  WARNING: Rotating root certificates will incur downtime between any services as they transition their mTLS certificate from one issuer to the next.

The self-signed root certificate, which is created via the Tresor package within OSM, will expire in a decade. To rotate the root certificate, the following steps should be followed:

1. Delete the `osm-ca-bundle` certificate in the OSM namespace
   ```console
   export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
   kubectl delete secret osm-ca-bundle -n $osm_namespace
   ```

2. Restart the control plane components
   ```console
   kubectl rollout restart deploy osm-controller -n $osm_namespace
   kubectl rollout restart deploy osm-injector -n $osm_namespace
   kubectl rollout restart deploy osm-bootstrap -n $osm_namespace
   ```

When the components gets re-deployed, you should be able to eventually see the new `osm-ca-bundle` secret in `$osm_namespace`:

```console
kubectl get secrets -n $osm_namespace
```

```
NAME                           TYPE                                  DATA   AGE
osm-ca-bundle                  Opaque                                3      74m
```

The new expiration date can be found with the following command:

```console
kubectl get secret -n $osm_namespace $osm_ca_bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -noout -dates
```

For Envoy services and validation certificates to be rotated the data plane components must be restarted.

#### Other certificate providers
The process of rotating the root certificate will be different for certificate providers other than Tresor. For Hashicorp Vault and cert-manager.io, users will need to rotate the root certificate themselves outside of OSM.

### Certificate Rotation with MeshRootCertificate

>  WARNING:  This feature is currently in preview and requires the `EnableMeshRootCertificate` feature flag enabled in the MeshConfig.  

The `MeshRootCertificate` is introduced in OSM v1.3. This custom resource can be enabled by using the `EnableMeshRootCertificate` feature flag when deploying OSM.

```console
osm install --set="osm.featureFlags.enableMeshRootCertificate=true"
```

> Important: If OSM is installed with `EnableMeshRootCertificate` disabled, but the feature flag is then later enabled in the MeshConfig, the control plane components must be restarted. This allows the flag to be picked up and begin using the `MeshRootCertificate` for certificate management. There will be no change in the root certificate, but it will now be managed by the `MeshRootCertificate`. 

### MeshRootCertificate

Each `MeshRootCertificate` represents a certificate that can be used in the mesh. There can be only two `MeshRootCertificate`'s that are used in the mesh at a given time and there must always be one `MeshRootCertificate` in the **active** `role`.

The following are the possible `MeshRootCertificate role`s:

- `active`: The certificate is used to sign **and** validate certificates in the mesh. There must always be one active certificate.
- `passive`: The certificate is used to validate certificates in the mesh.
- `inactive`: The certificate is not used in the mesh.

The following is an example of the `MeshRootCertificate` that will be created. The `provider` field can be `tresor`, `certManager`, or `vault`.  Before using the providers for [CertManager](certificates.md#using-cert-manager) or [Vault](certificates.md#using-hashicorp-vault), additional pre-configuration steps are required.  Refer to the [API documentation](#TODO) on how to configure the `MeshRootCertificate` fields for each provider.

Run `kubectl get meshrootcertificate` to see the default `MeshRootCertificate`:

```yaml
apiVersion: config.openservicemesh.io/v1alpha2
kind: MeshRootCertificate
metadata:
  name: osm-mesh-root-certificate
  namespace: osm-system  # Replace osm-system with the namespace where OSM is installed
spec:
  role: active
  provider:
    tresor:
      ca:
        secretRef:
          name: osm-ca-bundle
          namespace: osm-system  # Replace osm-system with the namespace where OSM is installed
  spiffeEnabled: false
  trustDomain: cluster.local
```

The only field that can be modified after initial creation is the `role` field.  Other fields will need to be changed by creating a new MeshRootCertificate and going through the [rotation process](#meshrootcertificate-rotation). On a new install, the initial `MeshRootCertificate` will be created by OSM and the certificate will be in the **active** `role`. Fields such as `trustDomain` and `spiffeEnabled` can be configured by passing options to the install command:

```bash
osm install --set="osm.featureFlags.enableSPIFFE=true --set="osm.trustDomain=cluster.example"
```

### MeshRootCertificate Rotation

To rotate the root certificate a mesh, a second `MeshRootCertificate` (MRC) will be created.  The second `MeshRootCertificate` will be created in a `passive` role and rotated into the `active` role. Once the new certificate is `active`, the old certificate will be moved to the `passive` role then finally `inactive`.  

The full process for rotating the certificate is:

| Step  |  original MRC(1) role   |  new MRC(2) role    | Signing MRC  | Validating MRC |
| ----- | ----------------------- | ------------------- | ------------ | -------------- |
| 1     | active                  | (not created)       | mrc1         | mrc1           |
| 2     | active                  | passive             | mrc1         | mrc1 and mrc2  |
| 3     | active                  | active              | mrc1 or mrc2 | mrc1 and mrc2  |
| 4     | passive                 | active              | mrc2         | mrc1 and mrc2  |
| 5     | inactive                | active              | mrc2         | mrc2           |
| 5     | (deleted)               | active              | mrc2         | mrc2           |


Each step requires time for the certificate information to be propagated to all the components in the mesh. The amount of time required will depend on your mesh. 

To verify if a certificate has been propagated, you can review the OSM controller logs and optionally run the following commands. If both certificates are in use, then you will see two certificates from the following commands, otherwise you will see the `active` certificate.  

We are working on improving this experience as we move the `MeshRootCertificate` feature out of preview.

Example commands:

- logs:
  ```
  kubectl logs osm-controller-67f9fd585d-ccr4b | grep "in progress root certificate rotation"
  ```
- osm validation and mutating webhooks certificates:
  ```
  kubectl get validatingwebhookconfigurations.admissionregistration.k8s.io osm-validator-mesh-osm -ojson | jq -r '.webhooks[] | .clientConfig.caBundle' | base64 --decode
  kubectl get mutatingwebhookconfigurations.admissionregistration.k8s.io osm-validator-mesh-osm -ojson | jq -r '.webhooks[] | .clientConfig.caBundle' | base64 --decode
  ```
- service certificates: 
  ```osm proxy get config_dump bookbuyer-b8c7bc4d9-zpm8m -n bookbuyer | jq -r '.configs[] | select(."@type"=="type.googleapis.com/envoy.admin.v3.SecretsConfigDump") | .dynamic_active_secrets[] | select(.name == "root-cert-for-mtls-outbound:bookstore/bookstore").secret.validation_context.trusted_ca.inline_bytes' | base64 -d
  ```
- xds certificates:
  ```
  osm proxy get config_dump bookbuyer-b8c7bc4d9-zpm8m -n bookbuyer | jq -r '.configs[] | select(."@type"=="type.googleapis.com/envoy.admin.v3.SecretsConfigDump") | .dynamic_active_secrets[] | select(.name == "validation_context_sds").secret.validation_context.trusted_ca.inline_bytes' | base64 -d
  ```

There is an experimental CLI command that can automate this process using a time-based method. Check out the [certificate rotation demo](../../demos/certificate_rotation.md) for more information. Please provide feedback through the [OSM GitHub repository](https://github.com/openservicemesh/osm) on the rotation process to help improve the experience.

#### Rotating with OSM's Tresor certificate

Tresor requires no additional configuration to rotate the certificates.  Create a second `MeshRootCertificate` and move it through the [rotation process](#meshrootcertificate-rotation).  OSM will create a secret for the values in `secretRef` and store the certificate there.

#### Rotating with Hashicorp Vault

Root rotation for Hashicorp Vault requires Vault version 1.11.  Learn how to rotate a root certificate in Vault through the [Hashicorp documentation](https://learn.hashicorp.com/tutorials/vault/pki-engine#step-7-rotate-root-ca).

Once the root certificate is rotated in Hashicorp you can configure a `MeshRootCertificate`, then move it through the [rotation process](#meshrootcertificate-rotation):

```yaml
apiVersion: config.openservicemesh.io/v1alpha2
kind: MeshRootCertificate
metadata:
  name: osm-mesh-root-certificate-2
  namespace: osm-system  # Replace osm-system with the namespace where OSM is installed
spec:
  role: passive
  provider:
    vault:
      host: vault.osm-system.svc.cluster.local
      port: 8200
      protocol: http
      role: <name of new Vault role>
      token:
        secretKeyRef:
          key: "osm-key"
          name: "osm-vault-token"
          namespace: osm-system  # Replace osm-system with the namespace where OSM is installed
  spiffeEnabled: false
  trustDomain: cluster.local
```

#### Rotating with cert-manager

Create a new issuer following the [cert-manager documentation](https://release-v1-2.docs.openservicemesh.io/docs/guides/certificates/#configure-cert-manger-for-osm-signing). 

Once you have a new issuer you can configure a `MeshRootCertificate` with the issuer information, then move it through the [rotation process](#meshrootcertificate-rotation):

```yaml
apiVersion: config.openservicemesh.io/v1alpha2
kind: MeshRootCertificate
metadata:
  name: osm-mesh-root-certificate-2
  namespace: osm-system  # Replace osm-system with the namespace where OSM is installed
spec:
  role: passive
  provider:
    certManager:
      	IssuerName:  <name>,
		IssuerKind:  "Issuer",
		IssuerGroup: "cert-manager.io",
  spiffeEnabled: false
  trustDomain: cluster.local
```
