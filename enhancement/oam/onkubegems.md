# kubevela on kubegems

## 集成

在 kubevela 的分析中，kubvela 可以分为：

- vela-core，运行于单集群的 oam 实现。
- cluster-gateway，提供多集群代理功能的 kubevela。
- vela-apiserver + velaux，提供 kubevela 界面和界面后端，以及多集群的插件安装等管理功能。

对于 kubegems 来说，仅需要 vela-core 即可,需要在每个集群上都运行一个 vela-core.

现在的 vela-core 就仅一个 controller deployment。

## 改动

使用 kubevela 后：

添加集群：

- 需要为每个集群都安装 vela-core 组件

对于开发用户（应用）：

- 应用就是创建到了环境中的 OAM application 资源，会被 render 和 apply。
- 移除应用编排功能。改为支持直接在环境中创建 application，支持在不同环境中执行 COPY，

  - 由于直接创建 oam application ，则前端需要重新开发支持 oam application 的 UI 页面。
  - 由于 application 中的各个 workload、trait 参数不同，但好在有 openAPIv3Schema 提供，可以根据 openAPIv3Schema 来生成配置的 UI。
  - 界面部分可以参考 velaux 或许可以复用。

- 由于 kubevela 完成了 apply 部分工作，现有的 argocd 功能也就不再需要了。

  - 由于移除了 argocd ，资源树功能会受到影响。但好在 kubevela 提供了 resourcetracer 功能，可以通过它获取到资源树。
  - 由于移除了 argocd ，部署历史功能会受到应用。但可以读取 application 的部署历史来替代。也更符合部署历史的设计。
  - 由于移除了 argocd ，镜像安全不能直接读取使用到的镜像，但可以通过解析 resourcetracer 中记录的资源来获取到使用的镜像。

- 由于 kubevela 直接操作 application 资源，最终的用户也会直接操作 application 资源。一个应用就只有一个 application 资源。应用编排功能也就不再需要了。

  - 由于移除了 应用编排（编辑）功能，git 服务也不再需要了。
  - 由于移除了 git，历史版本、回滚、diff 功能受到影响。但可以使用 applicationreversion 功能来替代。applicationreversion 记录了 application 的历史版本完整信息。

- 由于移除了 git argocd，异步任务功能也就不再需要了。
- 由于移除了 git argocd，应用商店下发不再使用 argocd。转而使用 kubevela 的 helm 类型 workload。
- 由于没有直接编辑 manifests，灰度发布功能无法使用。

  - 作为灰度发布功能的替代，可以使用 kubevela 提供的灰度发布功能（详细看后续部分）。
  - 使用 kubevela 灰度功能时，由于是和 application 紧密结合所以回滚等操作也会比较轻量。

**由于使用 kubevela 会对现有的应用中心做出较大改动，考虑以一个新的应用中心功能来进行开发，用户界面也可以重新设计。**

对于运维用户：

- 由于应用中心功能主要提供给开发用户使用，则对于运维用户需要设计单独的功能。
- 对于**每个集群**需要增加 “运维中心” 功能，运维中心提供与应用无关的环境独立配置。

  - 运维中心用于创建/配置运维特征 。例如： 默认存储类，默认存储卷大小。
  - 运维中心主要用于操作 OAM 中的 Traitdefinitions 资源。

- 环境下也需要增加 “运维中心” 功能，用于配置环境特异的配置，新增配置或覆盖默认配置。

## kubevela 解决的问题

### OAM

kubevela 帮助解决开发运维关注点分离的问题。

### service port protocol

在 OAM 中，service protocol 是原生支持的。

### 接入中心

原接入中心主要用于快速配置 监控、日志、追踪等，在 OAM 中，这些配置会作为应用侧运维特征被提供。

以 mysql exportter 举例：

- 在 OAM application 中，需要增加 mysql 类型 workload
- 在 Traitdefinition 中，若存在 mysql 类型 trairt 自动为其创建 mysql exporter。

职责分离的模式会让整个集成更加统一，更加方便。

## kubevela 缺失的功能

### statefulset

statefulset 目前在 OAM 上需要通过编写原生资源来实现。但我们可以编写 statefulset 的 workload 扩展来实现。

### 状态监控

虽然 kubevela 现在能够支持状态回填。但状态检查成功后，手动删除 wokload 状态不会再改变。

不过，这点可以增强 kubevela 的资源追踪功能来实现。资源变动后，可以快速更新 workload 的状态。需要一定的开发量。

### rollout

kubevela 的 [灰度发布和扩缩容](https://kubevela.io/zh/docs/end-user/traits/rollout) 目前暂时支持原生模式，通过控制两个 Deployment 调整副本比例来控制灰度。

这部分暂时还不太了解，似乎社区已经有基于 flux cd 的灰度实现，kubegems 仅做集成也可以。
