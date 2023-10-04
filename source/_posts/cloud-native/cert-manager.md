---
title: 基于 cert-manage 对阿里云域名自动签发 TLS(https) 证书
tags:
  + Kubernetes
  + TLS
categories: 云原生
keywords: 'blog, golang, php, k8s, kubernetes, tls, https'
description: cert-manager 是 Kubernetes 上的证书管理工具，可以从各种来源颁发证书，而且确保证书有效且最新，并尝试在到期前的配置时间续订证书。
cover: >-
  https://graph.linganmin.cn/231004/bf4bff6c40c7ab4fc47e037b810b58cb?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/231004/bf4bff6c40c7ab4fc47e037b810b58cb?x-oss-process=image/format,webp/quality,q_20
abbrlink: 2f918798
date: 2023-10-04 20:10:34
---

## 概述

使用 `HTTPS` 需要向权威机构申请证书，并且需要付出一定的成本，如果需求数量多，则开支也相对增加。 `cert-manager` 是 `Kubernetes` 上的全能证书管理工具，支持利用 `cert-manager` 基于 `ACME` 协议与 `Let's Encrypt` 签发免费证书并为证书自动续期，实现永久免费使用证书。

### 工作原理

![cert-manager原理](https://graph.linganmin.cn/231004/bf4bff6c40c7ab4fc47e037b810b58cb?x-oss-process=image/format,webp/quality,q_60)

### 证书签发

`Let's Encrypt` 是一个非营利性的证书颁发机构（Certificate Authority，简称 CA），旨在提供免费的 `SSL/TLS` 证书，以帮助网站和网络应用实现加密通信。SSL/TLS 证书是一种数字证书，用于加密数据传输以确保数据在客户端和服务器之间的安全性和隐私性。通过使用 `SSL/TLS` 证书，网站可以实现 `HTTPS` 连接，从而保护用户的数据免受窃听和篡改的威胁。

以下是 `Let's Encrypt` 的一些关键特点和信息：

1. **免费证书**: `Let's Encrypt` 提供免费的 SSL/TLS 证书，使网站所有者能够轻松地将其网站升级为安全的 HTTPS 连接，无需支付昂贵的证书费用。
2. **自动化**: `Let's Encrypt` 的证书管理工具和客户端使证书的获取和更新自动化，减少了手动证书管理的复杂性。
3. **开放性**: `Let's Encrypt` 的标准和协议是开放的，并且可以由任何人使用。这有助于推动更广泛的 HTTPS 采用，提高互联网的安全性。
4. **广泛支持**: `Let's Encrypt` 证书受到许多 Web 服务器和操作系统的支持，包括Apache、Nginx、Certbot、ACME 协议等。这使得在各种托管环境中使用 `Let's Encrypt` 证书变得更加容易。
5. **自我维护**: `Let's Encrypt` 证书有一个较短的有效期（通常为 90 天），但可以通过自动续订来保持有效。这鼓励了证书的定期更新，提高了安全性。
6. **支持多域名证书**: `Let's Encrypt` 证书支持多个域名（SAN 证书），允许多个域名共享同一个证书。

#### DNS 验证

`DNS-01` 和 `HTTP-01` 是 `Let's Encrypt` 证书颁发机构用于验证域名所有权的两种不同的方法，以下是这两种验证方法的简要说明：

1. `DNS-01`验证：
   - `DNS-01` 验证是一种基于域名系统（DNS）的验证方法，它要求域名所有者在 DNS 记录中添加一个特定的 `TXT记录` ，该记录包含一个随机的令牌或密钥。 `Let's Encrypt` 会查询DNS记录以确认令牌的存在，从而验证域名所有权。
   - 优点： `DNS-01` 验证不需要将任何文件或代码添加到您的网站服务器上，因此适用于不同类型的服务器和托管环境。
   - 缺点：配置 `DNS` 记录可能需要一些时间，因此在证书颁发之前可能需要等待一段时间。

2. `HTTP-01`验证：
   - `HTTP-01` 验证是一种基于 `HTTP` 协议的验证方法，它要求域名所有者在其网站根目录下放置一个特定的验证文件。 `Let's Encrypt` 会向您的网站发出一个特定的 `HTTP` 请求，以确认验证文件的存在。
   - 优点： `HTTP-01` 验证相对简单，通常能够快速完成，因为只需要在网站目录中添加一个文件。
   - 缺点：如果您的网站无法通过 `HTTP` 提供文件，或者如果网站是由第三方托管的，可能会导致验证失败。

选择 `DNS-01` 验证还是 `HTTP-01` 验证通常取决于您的具体需求和托管环境。如果您有完全控制域名的 `DNS` 记录，并且可以轻松地进行配置更改，那么 `DNS-01` 验证可能是一个不错的选择。如果您需要快速获得证书或无法在网站根目录下添加文件，那么 `HTTP-01` 验证可能更适合。

***后续操作我们将基于 `DNS-01` 进行...***

## 操作步骤

### 安装 cert-manager

#### Installing with Helm

* Add the Helm repository

  

```bash
  helm repo add jetstack https://charts.jetstack.io
  ```

* Update your local Helm chart repository cache

  

```bash
  helm repo update
  ```

* Install cert-manager

  

```bash
  helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.13.1 \
    --set installCRDs=true
  ```

#### Uninstalling with Helm

  

```bash
  # 删除组件
  helm --namespace cert-manager delete cert-manager
  # 删除 namespace
  kubectl delete namespace cert-manager
  ```

### 安装 alidns-webhook

由于 `cert-manager` 不支持 `AliDNS` ，所以我们只能以 `webhook` 方式来扩展 `DNS` 供应商。 `cert-manager` 为我们推荐了两个已适配 `AliDNS` 的开源项目，分别是[AliDNS-Webhook](https://github.com/pragkent/alidns-webhook)和[cert-manager-alidns-webhook](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook)，原理其实一样只是实现稍微不同，我们后续均使用[AliDNS-Webhook](https://github.com/pragkent/alidns-webhook)进行操作。

```bash
# Install alidns-webhook to cert-manager namespace. 
# 可能会因为网络环境问题会比较慢
kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
```

### 为 webhook 创建 alidns api secret

#### RAM 授权设置

- 创建RAM用户。
  - 使用阿里云账号登录RAM控制台。
  - 在访问控制控制台左侧导航栏，选择**身份管理 > 用户**。
  - 在用户页面，单击**创建用户**。
  - 在创建用户页面，输入登录名称和显示名称，**选中OpenAPI调用访问**，然后单击确定。记录RAM用户的`AccessKey ID`和`AccessKey Secret`。
- 授予新创建的RAM用户`AliyunDNSFullAccess`策略。
  - 在用户页面，单击上文创建的RAM用户右侧操作列下的**添加权限**。
  - 在添加权限面板系统策略下，输入`AliyunDNSFullAccess`，单击`AliyunDNSFullAccess`，单击确定。
  - 授予RAM用户自定义策略。
  - 在访问控制控制台左侧导航栏，选择**权限管理 > 权限策略**。
  - 在权限策略页面，单击**创建权限策略**。
  - 在创建权限策略页面，单击脚本编辑页签，然后输入以下内容，单击下一步。

    ```json
    {
        "Version": "1",
        "Statement": [
            {
                "Action": "*",
                "Resource": "acs:alidns:*:*:domain/<这里替换为你的域名>",
                "Effect": "Allow"
            },
            {
                "Action": [
                    "alidns:DescribeSiteMonitorIspInfos",
                    "alidns:DescribeSiteMonitorIspCityInfos",
                    "alidns:DescribeSupportLines",
                    "alidns:DescribeDomains",
                    "alidns:DescribeDomainNs",
                    "alidns:DescribeDomainGroups"
                ],
                "Resource": "acs:alidns:*:*:*",
                "Effect": "Allow"
            }
        ]
    }
    ```

  - 输入权限策略的名称，单击确定

#### Base64 编码

```bash
echo -n <AccessKey ID> | base64

echo -n <AccessKey Secret>  | base64
```

#### 资源清单

```yaml
# alidns-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
data:
  access-key: xxxxx  # Base64 编码后的 AccessKey ID。
  secret-key: xxxxx # Base64 编码后的 AccessKey Secret。
```

#### 创建 Secret

```bash
kubectl apply -f alidns-secret.yaml
```

### 部署 ClusterIssuer

#### ClusterIssuer 资源清单

```yaml
# clusterIssuer.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-issuer # 自定义个名字
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-issuer # 自定义个名字
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com # 这个不要改，在 AliDNS-Webhook 里写死了
          solverName: alidns # 这个固定写 alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret # 上面 alidns-secret.yaml 中的名字
              key: access-key # 上面 alidns-secret.yaml 中的对应数据的 key，下同
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key

```

#### 创建 ClusterIssuer

```bash
kubectl apply -f clusterIssuer.yaml
```

### 部署 Certificate

#### Certificate 资源清单

```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls # 自定义个名字
  namespace: default # 此处写你要使用证书的命名空间
spec:
  dnsNames:
    - "api.example.com" # 要签发证书的域名，替换成你自己的
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-issuer # 引用 ClusterIssuer，名字和 clusterIssuer.yaml 中保持一致
  secretName: example-tls # 最终签发出来的证书会保存在这个 Secret 里面
  duration: 2160h # 90d
  renewBefore: 360h # 15d

```

#### 创建 Certificate

```bash
kubectl apply -f certificate.yaml
```

## 在 Ingress 中使用证书

```yaml
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata:
#   name: kuard
#   annotations: {}
# spec:
#   ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com # 已签发证书的域名
    secretName: example-tls # 使用 certificate.yaml 存储签发证书的 Secret
  # rules:
  # - host: api.example.com
  #   http:
  #     paths:
  #     - path: /
  #       pathType: Prefix
  #       backend:
  #         service:
  #           name: kuard
  #           port:
  #             number: 80

```
