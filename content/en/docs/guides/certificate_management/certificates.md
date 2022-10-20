---
title: "Certificate Management"
description: "OSM uses mTLS for encryption of data between pods as well as Envoy and service identity."
type: docs
weight: 10
---

# Certificate Management

## mTLS and Certificate Issuance

Open Service Mesh uses mTLS for encryption of data between pods as well as Envoy and service identity. Certificates are created and distributed to each Envoy proxy via the SDS protocol by the OSM control plane.

## Types of Certificates

There are a few kinds of certificates used in OSM:

| Certificate Type | How it is used                                                                                | Validity duration                                                                              | Sample CommonName                                             |
| ---------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| xDS bootstrap    | used for Envoy-to-controller gRPC connection; identifies Envoy (Pod) to the xDS control plane | a decade                                                                                       | `7b2359d7-f201-4d3f-a217-73fd6e44e39b.bookstore-v2.bookstore` |
| service          | used for east-west communication between Envoys; identifies Service Accounts                  | default `24h`; defined by `osm.certificateProvider.serviceCertValidityDuration` install option | `bookstore-v2.bookstore.cluster.local`                        |
| webhook server   | used by the mutating, validating and CRD conversion webhook servers                           | a decade                                                                                       | `osm-injector.osm-system.svc`                                 |
|                  |


### Root Certificate

The root certificate for the service mesh is stored in an Opaque Kubernetes Secret named `osm-ca-bundle` in the namespace where osm is installed (by default `osm-system`).
The secret YAML has the following shape:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
      name: osm-ca-bundle
      namespace: osm-system
data:
  ca.crt: <base64 encoded root cert>
  private.key: <base64 encoded private key>
```

To read the root certificate (with the exception of Hashicorp Vault), you can retrieve the corresponding secret and decode it:

```console
kubectl get secret -n $osm_namespace $osm_ca_bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -text -noout
```

*Note: By default, the CA bundle is named `osm-ca-bundle`.*

This will provide valuable certificate information, such as the expiration date and the issuer.

#### Root Certificate Rotation

With OSM v1.3 root certificate rotation is in preview.  Refer to [certificate rotation](certificate-rotation.md) for more details.

## Issuing Certificates

Open Service Mesh supports 3 methods of issuing certificates:

- using an internal OSM package, called [Tresor](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor). This is the default for a first time installation.
- using [Hashicorp Vault](https://www.vaultproject.io/)
- using [cert-manager](https://cert-manager.io)

### Using OSM's Tresor certificate issuer

Open Service Mesh includes a package, [tresor](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor). This is a minimal implementation of the `certificate.Manager` interface. It issues certificates leveraging the `crypto` Go library, and stores these certificates as Kubernetes secrets.

- To use the `tresor` package during development set `export CERT_MANAGER=tresor` in the `.env` file of this repo.

- To use this package in your Kubernetes cluster set the `CERT_MANAGER=tresor` variable in the Helm chart prior to deployment.

Additionally:

- `osm.caBundleSecretName` - this string is the name of the Kubernetes secret, where the CA root certificate and private key will be saved.

### Using Hashicorp Vault

Service Mesh operators, who consider storing their service mesh's CA root key in Kubernetes insecure have the option to integrate with a [Hashicorp Vault](https://www.vaultproject.io/) installation. In such scenarios a pre-configured Hashi Vault is required. Open Service Mesh's control plane connects to the URL of the Vault, authenticates, and begins requesting certificates. This setup shifts the responsibility of correctly and securely configuring Vault to the operator.

The following configuration parameters will be required for OSM to integrate with an existing Vault installation:

- Vault address
- Vault token
- Validity period for certificates

`osm install` set flag control how OSM integrates with Vault. The following `osm install` set options must be configured to issue certificates with Vault:

- `--set osm.certificateProvider.kind=vault` - set this to `vault`
- `--set osm.vault.host` - host name of the Vault server (example: `vault.contoso.com`)
- `--set osm.vault.protocol` - protocol for Vault connection (`http` or `https`)
- `--set osm.vault.role` - role created on Vault server and dedicated to Open Service Mesh (example: `openservicemesh`)
- `--set osm.certificateProvider.serviceCertValidityDuration` - period for which each new certificate issued for service-to-service communication will be valid. It is represented as a sequence of decimal numbers each with optional fraction and a unit suffix, ex: 1h to represent 1 hour, 30m to represent 30 minutes, 1.5h or 1h30m to represent 1 hour and 30 minutes.

The Vault token must be provided to OSM so it can connect to Vault. The token can be configured as a set option or stored in a Kubernetes secret in the namespace of the OSM installation. If the `osm.vault.token` option is not set, the `osm.vault.secret.name` and `osm.vault.secret.key` options must be configured.

- `--set osm.vault.token` - token to be used by OSM to connect to Vault (this is issued on the Vault server for the particular role)
- `--set osm.vault.secret.name` - the string name of the Kubernetes secret storing the Vault token
- `--set osm.vault.secret.key` - the key of the Vault token in the Kubernetes secret

Additionally:

- `osm.caBundleSecretName` - this string is the name of the Kubernetes secret where the service mesh root certificate will be stored. When using Vault (unlike Tresor) the root key will **not** be exported to this secret.

#### Installing Hashi Vault

Installation of Hashi Vault is out of scope for the Open Service Mesh project. Typically this is the responsibility of dedicated security teams. Documentation on how to deploy Vault securely and make it highly available is available on [Vault's website](https://learn.hashicorp.com/vault/getting-started/install).

This repository does contain a [script (deploy-vault.sh)](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/demo/deploy-vault.sh), which is used to automate the deployment of Hashi Vault for continuous integration. This is strictly for development purposes only. Running the script will deploy Vault in a Kubernetes namespace defined by the `$K8S_NAMESPACE` environment variable in your [.env](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/.env.example) file. This script can be used for demonstration purposes. It requires the following environment variables:

```
export K8S_NAMESPACE=osm-system-ns
export VAULT_TOKEN=xyz
```

Running the `./demo/deploy-vault.sh` script will result in a dev Vault installation:

```
NAMESPACE         NAME                                    READY   STATUS    RESTARTS   AGE
osm-system-ns     vault-5f678c4cc5-9wchj                  1/1     Running   0          28s
```

Fetching the logs of the pod will show details on the Vault installation:

```
==> Vault server configuration:

             Api Address: http://0.0.0.0:8200
                     Cgo: disabled
         Cluster Address: https://0.0.0.0:8201
              Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.4.0

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://0.0.0.0:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: cZzYxUaJaN10sa2UrPu7akLoyU6rKSXMcRt5dbIKlZ0=
Root Token: xyz

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:
...
```

The outcome of deploying Vault in your system is a URL and a token. For instance the URL of Vault could be `http://vault.<osm-namespace>.svc.cluster.local` and the token `xxx`.
> Note: `<osm-namespace>` refers to the namespace where the osm control plane is installed.

#### Configure OSM with Vault

After Vault installation and before we use Helm to deploy OSM, the following parameters must be provided provided in the Helm chart:

```
CERT_MANAGER=vault
VAULT_HOST="vault.${K8S_NAMESPACE}.svc.cluster.local"
VAULT_PROTOCOL=http
VAULT_TOKEN=xyz
VAULT_ROLE=openservicemesh
```

When running OSM on your local workstation, use the following `osm install` set options:

```
--set osm.certificateProvider.kind="vault"
--set osm.vault.host="localhost"  # or the host where Vault is installed
--set osm.vault.protocol="http"
--set osm.vault.token="xyz"
--set osm.vault.role="openservicemesh'
--set osm.serviceCertValidityDuration=24h
```

#### How OSM Integrates with Vault

When the OSM control plane starts, a new certificate issuer is instantiated.
The kind of cert issuer is determined by the `osm.certificateProvider.kind` set option.
When this is set to `vault` OSM uses a Vault cert issuer.
This is a Hashicorp Vault client, which satisfies the `certificate.Manager`
interface. It provides the following methods:

```
  - IssueCertificate - issues new certificates
  - GetCertificate - retrieves a certificate given its Common Name (CN)
  - RotateCertificate - rotates expiring certificates
  - GetAnnouncementsChannel - returns a channel, which is used to announce when certificates have been issued or rotated
```

OSM assumes that a CA has already been created on the Vault server.
OSM also requires a dedicated Vault role (for instance `pki/roles/openservicemesh`).
The Vault role created by the `./demo/deploy-vault.sh` script applies the following configuration, which is only appropriate for development purposes:

- `allow_any_name`: `true`
- `allow_subdomains`: `true`
- `allow_baredomains`: `true`
- `allow_localhost`: `true`
- `max_ttl`: `24h`

Hashi Vault's site has excellent [documentation](https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine)
on how to create a new CA. The `./demo/deploy-vault.sh` script uses the
following commands to setup the dev environment:

    export VAULT_TOKEN="xyz"
    export VAULT_ADDR="http://localhost:8200"
    export VAULT_ROLE="openservicemesh

    # Launch the Vault server in dev mode
    vault server -dev -dev-listen-address=0.0.0.0:8200 -dev-root-token-id=${VAULT_TOKEN}

    # Also save the token locally so this is available
    echo $VAULT_TOKEN>~/.vault-token;

    # Enable the PKI secrets engine (See: https://www.vaultproject.io/docs/secrets/pki#pki-secrets-engine)
    vault secrets enable pki;

    # Set the max lease TTL to a decade
    vault secrets tune -max-lease-ttl=87600h pki;

    # Set URL configuration (See: https://www.vaultproject.io/docs/secrets/pki#set-url-configuration)
    vault write pki/config/urls issuing_certificates='http://127.0.0.1:8200/v1/pki/ca' crl_distribution_points='http://127.0.0.1:8200/v1/pki/crl';

    # Configure a role named "openservicemesh" (See: https://www.vaultproject.io/docs/secrets/pki#configure-a-role)
    vault write pki/roles/${VAULT_ROLE} allow_any_name=true allow_subdomains=true;

    # Create a root certificate named "osm.root" (See: https://www.vaultproject.io/docs/secrets/pki#setup)
    vault write pki/root/generate/internal common_name='osm.root' ttl='87600h'

The OSM control plane provides verbose logs on operations done with the Vault installations.

### Using cert-manager

[cert-manager](https://cert-manager.io) is another provider for issuing signed
certificates to the OSM service mesh, without the need for storing private keys
in Kubernetes. cert-manager has support for multiple issuer backends
[core](https://cert-manager.io/docs/configuration/) to cert-manager, as well as
pluggable [external](https://cert-manager.io/docs/configuration/external/)
issuers.

Note that [ACME certificates](https://cert-manager.io/docs/configuration/acme/)
are not supported as an issuer for service mesh certificates.

When OSM requests certificates, it will create cert-manager
[`CertificateRequest`](https://cert-manager.io/docs/concepts/certificaterequest/)
resources that are signed by the configured issuer.

### Configure cert-manger for OSM signing

cert-manager must first be installed, with an issuer ready, before OSM can be
installed using cert-manager as the certificate provider. You can find the
installation documentation for cert-manager
[here](https://cert-manager.io/docs/installation/).

Once cert-manager is installed, configure an [issuer
resource](https://cert-manager.io/docs/configuration/) to serve certificate
requests. It is recommended to use an `Issuer` resource kind (rather than a
`ClusterIssuer`) which should live in the OSM namespace (`osm-system` by
default).

Once ready, it is **required** to store the root CA certificate of your issuer
as a Kubernetes secret in the OSM namespace (`osm-system` by default) at the
`ca.crt` key. The target CA secret name can be configured on OSM using
`osm install --set osm.caBundleSecretName=my-secret-name` (typically `osm-ca-bundle`).

```bash
kubectl create secret -n osm-system generic osm-ca-bundle --from-file ca.crt
```

Refer to the [cert-manager demo](/docs/demos/cert-manager_integration) to learn more.

#### Configure OSM with cert-manager

In order for OSM to use cert-manager with the configured issuer, set the
following CLI arguments on the `osm install` command:

- `--set osm.certificateProvider.kind="cert-manager"` - Required to use cert-manager as the provider.
- `--set osm.certmanager.issuerName` - The name of the [Cluster]Issuer resource (defaulted to `osm-ca`).
- `--set osm.certmanager.issuerKind` - The kind of issuer (either `Issuer` or `ClusterIssuer`, defaulted to `Issuer`).
- `--set osm.certmanager.issuerGroup` - The group that the issuer belongs to (defaulted to `cert-manager.io` which is all core issuer types).
