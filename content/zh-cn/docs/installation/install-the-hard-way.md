---
title: "手动安装 Clusternet"
description: "如何手动安装 Clusternet。"
date: 2022-09-08
draft: false
weight: 3
---


本教程将引导您以“最硬核”的方式安装 Clusternet。 您也可以尝试参考 [使用Helm安装'Clusternet'](/docs/getting-started/install-with-helm)进行安装。

“最硬核”的方式安装 Clusternet 针对安装过程进行了优化，您可以一步一步的来了解 Clusternet，以确保您能理解每个必要的步骤，以便于确保成功安装 `Clusternet`。

---

您需要在子集群中部署`clusternet-agent`，在父集群中部署`clusternet-hub`和`clusternet-scheduler`。


{{% alert title="注意 🐳🐳🐳" color="primary" %}}
容器镜像同时托管在 [ghcr.io](https://github.com/orgs/clusternet/packages) 和 [dockerhub](https://hub.docker.com/u/clusternet).
您可自由选择，从更方便的镜像地址下载。
{{% /alert %}}

## 在父集群中部署 `clusternet-hub`

```bash
kubectl apply -f deploy/hub
```

接下来，您需要为群集注册创建一个 token。这将用于
`clusternet-agent`. `bootstrap token` 或 `service account token`都可以。

- 如果支持`bootstrapping`身份验证，即配置 `--enable-bootstrap-token-auth=true` `kube-apiserver` 将在父集群中运行，

  ```bash
  # 这将创建一个 bootstrap token 07401b.f395accd246ae52d
  kubectl apply -f manifests/samples/cluster_bootstrap_token.yaml
  ```

- 如果父集群中的 `kube-apiserver` 不支持`bootstrapping`身份验证 (例如 [k3s](https://k3s.io/))
  ,  `--enable-bootstrap-token-auth=false` (缺省为 `false`)，请改用`serviceaccount token`。

  ```bash
  # 这将创建一个 serviceaccount token
  kubectl apply -f manifests/samples/cluster_serviceaccount_token.yaml
  kubectl get secret -n clusternet-system -o=jsonpath='{.items[?(@.metadata.annotations.kubernetes\.io/service-account\.name=="cluster-bootstrap-use")].data.token}' | base64 --decode; echo
  # 这里将输出一个长字符串。请记住这一点。
  ```

## 在父集群中部署 `clusternet-scheduler`

```bash
kubectl apply -f deploy/scheduler
```

## 在子集群中部署 `clusternet-agent`

`clusternet-agent` 在子群集中运行，并帮助将自群集注册到父群集。

`clusternet-agent` 可以配置以下三种 `SyncMode` （通过参数 `--cluster-sync-mode`配置）,

- `Push` 表示父集群中的所有资源更改，都将由 `clusternet-hub` 自动同步、推送，并应用于子集群。
- `Pull` 表示 `clusternet-agent` 将监视、同步和应用父群集中的所有资源更改到子群集。
- `Dual` 结合了 `Push` 和 `Pull` 模式. 强烈建议使用此模式，通常和
  特性功能 `AppPusher` 一起使用。

特性功能  `AppPusher`  在代理端工作，主要出于以下两个原因引入，

- `SyncMode` 是不建议在注册后更改的, 这可能会导致设置和行为不一致。这就是为什么强烈建议使用 `Dual`。 当设置 `Dual` 模式时， `AppPusher` 提供帮助将 `Push` 模式切换到 `Pull`模式的方法，而无需更改参数 `--cluster-sync-mode`，反之亦然。

- 用于安全问题，例如子群集安全风险等。

  当子集群禁用特性功能 `AppPusher`，父集群不会向其部署任何应用程序，
  即使设置了同步模式 `Push` 或 `Dual` 。 此时，此子群集的工作方式类似于 `Pull` 模式。

  要部署的资源表示为 `Description`，您也可以运行自己的控制器来观察变更 `Description` 对象，然后分发和部署资源。

部署 `clusternet-agent`后，应首先创建包含群集注册token的机密。

```bash
# 创建命名空间 clusternet-system（如果未创建）
kubectl create ns clusternet-system
# 这里我们使用上面创建的token
PARENTURL=https://192.168.10.10 REGTOKEN=07401b.f395accd246ae52d envsubst < ./deploy/templates/clusternet_agent_secret.yaml | kubectl apply -f -
```

> :pushpin: :pushpin: Note:
>
> 如果您是通过service account token验证，请将 07401b.f395accd246ae52d 替换为之前生成的 token

上面的 `PARENTURL` 是您要注册到的父集群的 apiserver 地址， `apiserver` 地址必须以 `https` 开头。目前仅支持 `https` 。 如果 `apiserver` 服务器未侦听标准 https 端口 (:443), 请在 URL 中指定端口号，以确保代理连接到正确的端点，对于实例 `https://192.168.10.10:6443`。

```bash
# 如果需要选用其他同步模式，请修改以下 YAML 文件中的 SyncMode
kubectl apply -f deploy/agent
```

## 检查集群注册

请按照 [本指南](/zh-cn/docs/tutorials/cluster-management/checking-cluster-registration/) 检查集群注册状态。
