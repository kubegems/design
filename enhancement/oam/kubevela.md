# kubevela

[kubevela](https://kubevela.io) 是对 OAM 的实现，并且额外扩展了一些功能。包括：

- 工作流(Workflow)。可以理解为 CD 工具，由于 OAM 中未定义”部署“动作，工作流被用于完成最终的部署操作。
- 应用策略（Policy)。应用策略是 OAM 中的 ScopeDefination 。

非 OAM 的功能有：

- 集群（Cluster），支持对多个集群进行 OAM 应用部署。
- 插件（Addon），支持使用插件的方式扩展 OAM 和 kubevela 的功能。

## 安装

```sh
helm repo add kubevela https://charts.kubevela.net/core
helm repo update
helm install --create-namespace -n vela-system kubevela kubevela/vela-core --version 1.3.3
```

## 组件

[kubevela-architecture](https://kubevela.net/zh/docs/getting-started/architecture)

- core,核心组件。运行着许多个 controller 。实际处理 Application 资源的部分。
- [cluster-gateway](https://github.com/oam-dev/cluster-gateway), 用于将请求路由到不同集群，与 clustergateway 资源关联。基于 sample-apiserver 实现。
- vela-apiserver, 用于为 velaux 界面提供后端 API。用于读取 OAM 资源扩展信息，并且提供给 vela-ui 用于展示。
- [velaux](https://github.com/kubevela/velaux)

## Core

core 部分包含 kubeleva 主要逻辑：

- application controller，用于管理应用生命周期。例如 应用渲染，部署，删除，状态回填。
- traitdefinition controller，用于管理 traitdefinition 资源。生成历史版本和将其存储至 configmap 中。
- componentdefinition controller，用于管理 componentdefinition 资源。生成历史版本和将其存储至 configmap 中。
- policydefinition controller，用于管理 policydefinition 资源。生成历史版本和将其存储至 configmap 中。
- workflowstepdefinition controller，用于管理 workflowstepdefinition 资源。生成历史版本和将其存储至 configmap 中。

从上面可以分析到 kubevela 的主要逻辑仅在 application controller 中：

- 应用渲染，渲染 workload，渲染 trait，渲染 policy。
- 应用部署，部署渲染完成的 workload。
- 状态检查，通过 cue 模板中指定的状态检查，检查应用是否满足指定的状态。状态检查支持 workload 和 trait。
- 资源追踪，ResourceTracker 会记录当前 application 所创建的资源。（似乎不会实时更新）
- 垃圾回收，GarbageCollector 删除过期的应用版本
