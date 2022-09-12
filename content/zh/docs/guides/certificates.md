---
title: "证书管理"
description: "OSM 使用 mTLS 为 Pod 间的数据加密，就像 Envoy 和服务标识那样"
type: docs
weight: 10
---

# 证书管理

## mTLS 和证书颁发

开放式服务网格使用 mTLS 来为 Pod 间的数据加密，就像 Envoy 和服务标识那样。证书被创建然后分布到每个 Envoy 代理，这个由 OSM 控制平面通过 SDS 协议来完成。

## 证书的类型

这里是在 OSM 里被使用的一些证书种类：

| 证书类型 | 如何被使用                                                                                | 有效期限                                                                              | 通用名示例                                             |
| ---------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| xDS 引导    | 为 Envoy-to-controller gRPC 连接所用; 标识 Envoy (Pod) 到 xDS control plane | 十年                                                                      | `7b2359d7-f201-4d3f-a217-73fd6e44e39b.bookstore-v2.bookstore` |
| 服务          | 为 Envoy 之间的 east-west 通信所用; 标识服务账号                  | 默认 `24h`; 被 `osm.certificateProvider.serviceCertValidityDuration` 安装选项所定义 | `bookstore-v2.bookstore.cluster.local`                        |
| webhook 服务器   | 被突变、验证和 CRD 转换 webhook 服务器所使用                           | 十年                                                                                       | `osm-injector.osm-system.svc`                                 |
|                  |


### 根证书

服务网格的根证书被存储在一个名为 `osm-ca-bundle` 的 Opaque Kubernetes Secret 里，其在 OSM 被安装的命名空间里 (默认是 `osm-system`)。
Secret YAML 有如下的格式：

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
      name: osm-ca-bundle
      namespace: osm-system
data:
  ca.crt: <base64 encoded root cert>
  expiration: <base64 encoded ISO 8601 certificate expiration date; for example: 2031-01-11T23:15:09.299Z>
  private.key: <base64 encoded private key>
```

欲了解细节和被使用的代码，请参阅[osm-controller.go](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/cmd/osm-controller/osm-controller.go#L182-L183)。

要阅读根证书 (除 Hashicorp Vault 外)，可以获取相应的 secret 然后解码它：

```console
kubectl get secret -n $osm_namespace $osm_ca_bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -text -noout
```

*注意：默认的，CA 包被命名为 `osm-ca-bundle`。*

这个将提供有价值的证书信息，例如过期日期和发行者。

### 轮换根证书 (Tresor)

自签名根证书，其通过 OSM 内的 Tresor 包来创建，将在一个十年后过期。要轮换该根证书，下面的步骤应该被遵循：

1. 删除在 OSM 命名空间里的 `osm-ca-bundle` 证书
   ```console
   export osm_namespace=osm-system # Replace osm-system with the namespace where OSM is installed
   kubectl delete secret osm-ca-bundle -n $osm_namespace
   ```

2. 重启 Control Plane 组件
   ```console
   kubectl rollout restart deploy osm-controller -n $osm_namespace
   kubectl rollout restart deploy osm-injector -n $osm_namespace
   kubectl rollout restart deploy osm-bootstrap -n $osm_namespace
   ```

当组件得到了重新部署，应该可以在 `$osm_namespace` 里面最终看到新的 `osm-ca-bundle` secret。

```console
kubectl get secrets -n $osm_namespace
```

```
NAME                           TYPE                                  DATA   AGE
osm-ca-bundle                  Opaque                                3      74m
```

新的过期日期可以通过下面的命令来获取：

```console
kubectl get secret -n $osm_namespace $osm_ca_bundle -o jsonpath='{.data.ca\.crt}' |
    base64 -d |
    openssl x509 -noout -dates
```

要轮换 sidecar 的服务和验证证书，必须重新启动数据平面组件。

#### Hashicorp Vault 和 Certmanager

对于 Tresor 之外的证书提供商，轮换根证书的过程将会是不同的。对于 Hashicorp Vault 和 cert-manager.io，用户将需要在 OSM 之外自己轮换根证书。

## 颁发证书

开放式服务网格支持 3 种方法来发布证书：

- 使用一个内部 OSM 包，其被称为 [Tresor](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor)。对于第一次安装来说，这是默认的方法。
- 使用 [Hashicorp Vault](https://www.vaultproject.io/)
- 使用 [cert-manager](https://cert-manager.io)

### 使用 OSM 的 Tresor 证书发行者

开放式服务网格包含一个包，[tresor](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/pkg/certificate/providers/tresor)。这是一个 `certificate.Manager` 接口的最小实现。它利用 `crypto` Go 库来发行证书，并把这些证书作为 Kubernetes secret 来存储。

- 要在开发期间使用 `tresor` 包，在 repo 的 `.env` 文件中设置 `export CERT_MANAGER=tresor`。

- 在 Kubernetes 集群中要使用这个包，在之前部署的 Helm chart 中设置 `CERT_MANAGER=tresor` 变量。

此外：

- `osm.caBundleSecretName` —— 这个字符串是该 Kubernetes Secret 的名字，这里 CA 根证书和私钥将被保存。

### 使用 Hashicorp Vault

服务网格运维人员，那些认为把他们的服务网格的 CA 根证书存在 Kubernetes 不安全的人，可以有选择集成一个 [Hashicorp Vault](https://www.vaultproject.io/) 安装。在这样的场景下，一个预制的 Hashi Vault 被需要。开放式服务网格的 Control Plane 连接到该 Vault 的 URL，鉴权并且开始索取证书。对运维人员来说，这个安装升格了正确地和安全地配置 Vault 的响应能力。

如下配置参数将被需要来给 OSM 集成一个现成的 Vault 安装：

- Vault 地址
- Vault 令牌
- 证书的验证周期

`osm install` 设置标志来控制 OSM 如何来集成 Vault。接下来的 `osm install` 设置选项必须被配置来发放带 Vault 的证书：

- `--set osm.certificateProvider.kind=vault` —— 设置为 `vault`
- `--set osm.vault.host` —— Vault 服务器的主机名 (例如：`vault.contoso.com`)
- `--set osm.vault.protocol` —— Vault 连接的协议 (`http` 或者 `https`)
- `--set osm.vault.role` —— 角色创建在 Vault 服务器上并且专属于开放式服务网格 (例如：`openservicemesh`)
- `--set osm.certificateProvider.serviceCertValidityDuration` —— 每一个为服务到服务通信所发放的新证书有效的周期。它用一串十进制数带可选的小数和一个单位后缀来表达，例如 1h 表示 1 小时，30m 表示 30 分钟，1.5h 或者 1h30m 表示 1 小时 30 分钟。

为了 OSM 可以连接到 Vault，必须提供 Vault 令牌。令牌可以通过 set 选项配置，也可以存储在安装 OSM 的命名空间中的 Kubernetes secret中。如果没有设置 `osm.vault.token` 选项，则一定要设置 `osm.vault.secret.name` 和 `osm.vault.secret.key` 选项。

- `--set osm.vault.token` - OSM 用于连接 Vault 的令牌（这是由 Vault 服务器为特定角色颁发的）
- `--set osm.vault.secret.name` - 存储 Vault 令牌的 Kubernetes secret 名字
- `--set osm.vault.secret.key` - Kubernetes secret 中 Vault 令牌的键

此外：

- `osm.caBundleSecretName` 这个字符串是该 Kubernetes Secret 的名字，该服务网格的根证书将被存储在该 Secret。当使用 Vault (不同于 Tresor)，根键将**不会被**导出给这个 Secret。

#### 安装 Hashi Vault

Hashi Vault 的安装超出了开放式服务网格项目的范围。这属于专属的安全团队的响应能力。关于如何安全地部署 Vault 并使其高度可用的文档在 [Vault 网站](https://learn.hashicorp.com/vault/getting-started/install)。

这个仓库包含一个 [脚本 (deploy-vault.sh)](https://github.com/openservicemesh/osm/tree/{{< param osm_branch >}}/demo/deploy-vault.sh)，该脚本被用来为 CI 做 Hashi Vault 的自动化部署。它被严格地限制仅为开发目的。运行该脚本将部署 Vault 在一个 Kubernetes 命名空间里，该命名空寂被 [.env](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/.env.example) 文件里面的 `$K8S_NAMESPACE` 环境变量所定义。这个脚本能够以演示目的而使用。它需要下面的环境变量：

```
export K8S_NAMESPACE=osm-system-ns
export VAULT_TOKEN=xyz
```

运行 `./demo/deploy-vault.sh` 脚本将导致在一个 dev 里有 Vault 安装：

```
NAMESPACE         NAME                                    READY   STATUS    RESTARTS   AGE
osm-system-ns     vault-5f678c4cc5-9wchj                  1/1     Running   0          28s
```

提取 Pod 的 log 将显示关于 Vault 安装的细节：

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

在系统里面部署 Vault 的可以得到一个 URL 和一个令牌。Vault 的 URL 实例应该是 `http://vault.<osm-namespace>.svc.cluster.local` 和令牌 `xxx`。
> 注意：`<osm-namespace>` 引用了 OSM Control Plane 被安装的命名空间。

#### 配置带 Vault 的 OSM

Vault 安装之后并且在我们使用 Helm 部署 OSM 之前，下面的参数必须在 Helm chart 里被提供：

```
CERT_MANAGER=vault
VAULT_HOST="vault.${K8S_NAMESPACE}.svc.cluster.local"
VAULT_PROTOCOL=http
VAULT_TOKEN=xyz
VAULT_ROLE=openservicemesh
```

当在本地工作站运行 OSM，使用下面的 `osm install` 设置选项：

```
--set osm.certificateProvider.kind="vault"
--set osm.vault.host="localhost"  # or the host where Vault is installed
--set osm.vault.protocol="http"
--set osm.vault.token="xyz"
--set osm.vault.role="openservicemesh'
--set osm.serviceCertValidityDuration=24h
```

#### OSM 如何集成 Vault

当 OSM Control Plane 开始，一个新的证书发行者被实例化。
证书发行者的类型被 `osm.certificateProvider.kind` 设置选项来决定。
当这个被设置成 `vault`，OSM 使用一个 Vault 证书发行者。
这是一个 Hashicorp Vault 客户端，其满足 `certificate.Manager` 接口。它提供下面的方法：

```
  - IssueCertificate - issues new certificates
  - GetCertificate - retrieves a certificate given its Common Name (CN)
  - RotateCertificate - rotates expiring certificates
  - GetAnnouncementsChannel - returns a channel, which is used to announce when certificates have been issued or rotated
```

OSM 假设一个 CA 已经在 Vault 服务器上被创建。
OSM 也需要一个专属的 Vault 角色 (例如 `pki/roles/openservicemesh`)。
被 `./demo/deploy-vault.sh` 脚本创建的 Vault 角色被应用到下面的配置，该配置仅适合开发目的：

- `allow_any_name`: `true`
- `allow_subdomains`: `true`
- `allow_baredomains`: `true`
- `allow_localhost`: `true`
- `max_ttl`: `24h`

Hashi Vault 的站点有卓越的[文档](https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine)关于如何创建一个新的 CA。`./demo/deploy-vault.sh` 脚本用下面的命令来安装开发环境：

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

OSM 控制平面提供 Vault 安装详实的操作 log。

### 使用 cert-manager

[cert-manager](https://cert-manager.io) 是另一个提供者，用来颁发签名证书给 OSM 服务网格，不需要在 Kubernetes 里存储私钥。cert-manager 支持多个发行者后端[核心](https://cert-manager.io/docs/configuration/)，同样也有可插入的[外部](https://cert-manager.io/docs/configuration/external/)发行者。

注意作为对于服务网格证书的发行者，[ACME 证书](https://cert-manager.io/docs/configuration/acme/)不被支持。

当 OSM 请求证书时，它将创建 cert-manager [`CertificateRequest`](https://cert-manager.io/docs/concepts/certificaterequest/) 资源，该资源被配置的发行者签名。

### 为 OSM 签名配置 cert-manager

在 OSM 使用 cert-manager 作为证书提供者能够被安装之前，cert-manager 必须首先被安装，并带一个可用的发行者。可以在[这里](https://cert-manager.io/docs/installation/)找到 cert-manager 的安装文档。

一旦 cert-manager 被安装，配置一个[发行者资源](https://cert-manager.io/docs/configuration/)来服务证书请求。推荐使用一个 `Issuer` 资源类 (而不是一个 `ClusterIssuer`)，其应该居于 OSM 命名空间 (默认是 `osm-system`)。

一旦就绪，它被要求在 `ca.crt` 键上 OSM 命名空间 (默认是 `osm-system` ) 里存储发行者根 CA 证书，以作为一个 Kubernetes Secret。目标 CA Secret 名称能够在 OSM 上使用 `osm install --set osm.caBundleSecretName=my-secret-name` (典型的是 `osm-ca-bundle`) 来配置。

```bash
kubectl create secret -n osm-system generic osm-ca-bundle --from-file ca.crt
```

参考 [cert-manager 演示](/docs/demos/cert-manager_integration) 来了解更多。

#### 用 cert-manager 来配置 OSM

为了 OSM 能使用带配置的发行者的 cert-manager，在 `osm install` 命令上设置下面的 CLI 参数：

- `--set osm.certificateProvider.kind="cert-manager"` —— 需要使用 cert-manager 做个提供者。
- `--set osm.certmanager.issuerName` —— [集群] 发行者资源 (默认是 `osm-ca`) 的名字。
- `--set osm.certmanager.issuerKind` —— 发行者种类 (要么 `Issuer`，或者 `ClusterIssuer`，默认是 `Issuer`)。
- `--set osm.certmanager.issuerGroup` —— 发行者所属的组 (默认是 `cert-manager.io`，是所有核心发行者的类型)
