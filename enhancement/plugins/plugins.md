# 插件

实现更快速、灵活、可扩展的 kubegems 插件系统。

## 背景

kubegems 原有插件使用 ansible operator 方式实现，
由于安装过程中需要处理的东西较多。ansible 仅能串行执行，
初次安装和对安装配置执行更改时都需要重新运行整个 ansible role，从更改到成功完成修改需要数分钟到数十分钟。

## 需求

重构 ansible 安装方式最初需求为提高速度，包括初次安装和后续更改。

- 快速响应，最好能在数秒内完成 apply 动作，对修改响应迅速。
- 插件化，与 ansible inatller 相同，支持将不同功能的组件作为一个部分(插件)，可以独立控制开启/关闭。
- 可配置

对于镜像，需要支持所有组件能够使用自定义镜像，以便于在默认镜像仓库缓慢是使用自建仓库或其他三方仓库分发镜像，以加快安装进度。

- 全局支持定义镜像
- 兼容旧版本

## 设计

### 快速响应

Ansible installer 的主要用于将各资源 apply 到集群中，运行缓慢由于串行运行。
作为改善，可以直接使用 kubectl 方式直接 apply 到集群。对于没有依赖关系的部分可以并行执行。

直接调用 kubectl 的方式显然不够友好和可扩展，考虑使用 client-go 或其他相关库使用编码方式完成。

由于 installer 的各个插件都是独立三方插件，例如 prometheus loki cert-manager argo 等，
其部署文件可能随着版本升级而升级，且这些组件都有其官方维护的 helm chart，
使用 helm 方式部署这些组件既能够在版本更新时跟着官方快速升级，且`production ready`为生产环境设计，也无需自行维护部署文件。
对于使用者，使用来自组件官方的 helm chart 更为透明，接受度更高。

所以，在安装时使用 helm(替代 kubectl) 将 chart apply 到集群中的方式，即简单又快速。
除了有现成 helm chart 的组件，对于自有组件或者其他资源，可以制作为 helm chart 方式或使用其他 "apply" 方式执行。

其他 apply 方式可以为 kustomize 或者使用 gitops engine 进行 sync，使用 gitops engine 的方式可以方便的进行资源清理，patch 也更方便。

对于 kubegem 的中心组件和边缘组件，可以分别制作为独立的 helm chart。

### 插件化

插件化的主要功能为能够控制不同组件的开启/关闭。在 installer 中，可以使用一个 Plugin CR 来控制。
一个 Plugin 对应一个独立的 helm chart，可以通过修改 Plugin CR 来控制组件开关。
因此 Plugin 还需要一个 plugin controller 来处理 Plugin CR 的变化，控制 helm 执行安装/卸载操作。

如此，整个安装流程就变为多个 Plugin CR 的下发。多个 Plugin 如果存在依赖关系，可以在 Plugin 中定义依赖关系，仅在依赖的 Plugin 状态正常时才执行安装。
这个逻辑由 plugin controller 控制。

```yaml
apiVersion: plugins.kubegems.io/v1beta1
kind: Plugin
spec:
  enabled: true
  kind: helm
  repo: https://charts.example.com
  name: component-a
  version: v1.0.0
```

### 可配置

可配置的范围可以是整个集群，也可能是整个组件。

对于组件内的可配置，helm template 以及可以覆盖绝大部分的场景，例如配置 storage class，镜像 repository 等，都可以通过更改 helm chart value 的少量字段来完成。
在实现上为将 value 支持放入 Plugin 定义中，通过更改 Plugin .spec.value 中的内容，plugin controller 将其映射为 helm chart value 中对应的字段并自动完成更新。

```diff
apiVersion: plugins.kubegems.io/v1beta1
kind: Plugin
spec:
  enabled: true
  kind: helm
  repo: https://charts.example.com
  name: component-a
  version: v1.0.0
+ values:
+   subComponent:
+     enabled: false
+   storageClass:
+     name: local-path
```

对于整个集群的可配置，也可以借助相似的方式完成。
创建一个 allinone 的 “chart” ,其中的资源为所有 Plugin，对于插件的开/关，自定义都可以通过配置不同的 value 来控制某个 Plugin 如何渲染，从而实现控制整个集群的配置。

举例：

```go-template
# template/compoents-a.yaml
{{- if .Values.component-a.enabled }}
apiVersion: plugins.kubegems.io/v1beta1
kind: Plugin
spec:
  enabled: true
  kind: helm
  values:
    subComponent:
      enable: {{ .Values.component-a.subComponent.enabled }}
{{- end }}
```

```yaml
# values.yaml
component-a:
  enable: true
  subComponent:
    enabled: true
```

### 镜像本地化

大部分 helm chart 都提供了自定义镜像仓库的功能，只需要设置这些字段即可。

镜像离线/镜像复制，要将来自 docker.io quay.io gcr.io 等的镜像复制进第三方或者自建仓库时，需要快速提供 helm charts 中使用的原始镜像。

提取可以使用 helm template 并提取出其中的 `image:` 字段值, 可以编写适当的脚本完成。

镜像复制传统方式使用 pull tag push 方式，但还有更好的方式，使用 OCI 工具 `skopeo copy` 来完成而无需将所有镜像 layer 拉至本地,对于目标仓库中已有的 layer 可直接忽略。

示例：

```sh
skopeo copy docker://docker.io/library/helloworld:v1  docker://registry.example.com/library/helloworld:v1
```

### 向后兼容

总体上来说 plugins 本身带来了较大的变动，对于非必要的向后兼容，可以直接舍弃。

对于必要的向后兼容，通过增加可配置项进行。
例如 namespace， storage class ，pvc 容量等。
但由于使用 helm chart，旧版本无法平滑迁移，部分应用可能需要先删除旧版本再创建新版本。
