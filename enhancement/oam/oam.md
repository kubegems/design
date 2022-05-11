# oam 规范简述

开放应用程序模型 (OAM) 是一组标准但更高级别的抽象，用于在当今的混合和多云环境之上对云原生应用程序进行建模。

专注于应用程序而不是容器或编排器，开放应用程序模型带来了模块化、可扩展和可移植的设计，用于定义具有更高级别 API 的应用程序部署。
这是跨混合环境（包括 Kubernetes、云，甚至物联网设备）实现简单、一致且强大的应用程序交付的关键。

为什么要开放应用模型？

在当今的混合部署环境中，在没有应用程序上下文的情况下交付应用程序很困难：

- 开发人员将时间花在基础设施细节上而不是应用程序上——集群、入口、标签、DNS 等，并学习如何在不同环境中实现基础设施
- 不可扩展 - 可能会引入上层平台，但几乎可以肯定，您的应用程序的需求将很快超出该平台的功能。
- 供应商锁定 - 应用程序部署与服务提供商和基础设施紧密耦合，这会严重影响您在混合环境中配置、开发和操作应用程序的方式。

在开放应用程序模型中，我们提出了一种以应用程序为中心的方法：

- 应用程序优先 - 使用自包含模型定义应用程序部署，其中操作行为作为应用程序定义的一部分，无需基础架构，只需部署即可。
- 清晰性和可扩展性 - 一种开放标准，可将应用交付模块化为可重复使用的部分，根据您自己的需要将它们组装成部署计划，完全自助。
- 与供应商无关 - 一种一致但更高级别的抽象，用于对跨本地集群、云提供商甚至边缘设备的应用交付建模。零锁定。
- 开放式应用程序模型的设计由 KubeVela 项目驱动 - 一个现代应用程序部署平台，旨在使在当今的混合、多云环境中交付和管理应用程序更容易和更快。

## 设计原则

开放应用程序模型遵循一组设计原则，以确保模型的清晰性、丰富性和可扩展性。

### 关注点分离

关注点分离是参考正在解决的离散问题做出架构选择的设计理念。像“组件”和“应用程序”或“示意图”和“配置”这样的工件的划分是通过沿着功能或行为线划分的。通过识别不同用户组的角色和职责，规范被划分为与问题空间相匹配的概念。

### 运行时中立

开放应用程序模型与运行时无关。它不假定任何特定于运行时的功能。相反，它旨在为应用程序所有者和运营商提供一个通用词汇表来描述独立于任何特定平台的所需拓扑和行为。

### 平衡（优雅）

在确保关注点分离的同时，OAM 力求避免给在较小团队中担任多个角色的用户带来不必要的复杂性。简单的场景应该可以用最少的时间和精力投资来实现，但复杂的场景应该可以适应而不需要重新平台化。

OAM 提供了多个抽象层，因此可以独立于开发人员的关注点来捕获运营关注点。

### 可重用性

OAM 原理图中的组件设计为可重复使用和共享。此外，它们保持独立于它们描述的代码，从而可以重用代码（容器），并防止“锁定”情况。

该模型作为一个整体旨在提供应用程序“分布”，其中相同的应用程序可以在不同平台上执行而无需更改。应用程序的这种可移植性旨在使以下场景不仅成为可能，而且很容易：

- 将应用程序从开发人员工作站转移到生产集群或服务
- 从一种实现迁移到另一种实现，无需更改代码
- 创建可以将应用程序部署到客户平台上的类似市场的环境

### 应用程序模型不是编程模型

应用程序模型和编程模型之间有明显的区别。应用程序模型描述了应用程序的组成及其组件的拓扑结构。它不关心每个组件是如何实现的（语言、设计模式等）。

另一方面，编程模型描述了单个软件是如何组成的。开发人员使用它来实现应用程序组件。开放应用程序模型提供了一个对编程模型没有任何要求的应用程序模型。

## 开放应用模型

OAM 模型使用 Kubernetes API 资源约定进行描述。(使用 k8s 风格 api 也是为了在云原生中更好的实现和集成)

整个 OAM 模型由以下几个部分组成：

- ComponentDefiniton，组件定义。组件代表一个可运行的单元，以及一个描述（概要信息）。
- WorkloadDefiniton，工作负载定义。工作负载类型标识组件可以执行的不同工作负载。
- TraitDefinition，特征定义。特征是用额外的特定于操作的特性来扩充组件的叠加层。特征代表运营商的关注，而不是开发人员/软件所有者的关注。
- Application Scope，应用范围。应用程序范围通过将具有共同属性或依赖关系的组件分组来表示应用程序边界。
- Application，应用程序。应用程序配置组合了一组组件实例、它们的特征以及它们所在的应用程序范围，并结合了配置参数和元数据。

![overview](https://github.com/oam-dev/spec/raw/master/assets/overview.png)

简单来说就是，一个应用由 一个或者多个“组件”，运维“特征”，“范围”组成。

## 组件定义(ComponentDefinition)

组件描述了可以作为更大的分布式应用程序的一部分。ComponentDefinition 用于允许组件提供者以与基础设施无关的格式声明此类执行单元的运行时特性。

例如，应用程序中的每个微服务都被描述为一个组件，一个简单的容器化工作负载、Helm 或云数据库都可以建模为一个组件。

一个示例：

组件定义：

```yaml
apiVersion: core.oam.dev/v1beta1
kind: ComponentDefinition
metadata:
  name: webserver
  annotations:
    definition.oam.dev/description: "webserver is a combo of Deployment + Service"
spec:
  workload:
    definition:
      apiVersion: apps/v1
      kind: Deployment
  schematic:
    cue:
      template: |
        output: {
            apiVersion: "apps/v1"
            kind:       "Deployment"
            spec: {
                selector: matchLabels: {
                    "app.oam.dev/component": context.name
                }
                template: {
                    metadata: labels: {
                        "app.oam.dev/component": context.name
                    }
                    spec: {
                        containers: [{
                            name:  context.name
                            image: parameter.image

                            if parameter["cmd"] != _|_ {
                                command: parameter.cmd
                            }

                            if parameter["env"] != _|_ {
                                env: parameter.env
                            }

                            if context["config"] != _|_ {
                                env: context.config
                            }

                            ports: [{
                                containerPort: parameter.port
                            }]

                            if parameter["cpu"] != _|_ {
                                resources: {
                                    limits:
                                        cpu: parameter.cpu
                                    requests:
                                        cpu: parameter.cpu
                                }
                            }
                        }]
                }
                }
            }
        }
        // an extra template
        outputs: service: {
            apiVersion: "v1"
            kind:       "Service"
            spec: {
                selector: {
                    "app.oam.dev/component": context.name
                }
                ports: [
                    {
                        port:       parameter.port
                        targetPort: parameter.port
                    },
                ]
            }
        }
        parameter: {
            image: string
            cmd?: [...string]
            port: *80 | int
            env?: [...{
                name:   string
                value?: string
                valueFrom?: {
                    secretKeyRef: {
                        name: string
                        key:  string
                    }
                }
            }]
            cpu?: string
        }
```

组件使用：

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: webserver-demo
spec:
  components:
    - name: hello-world
      type: webserver # claim to deploy webserver component definition
      properties: # setting parameter values
        image: crccheck/hello-world
        port: 8000 # this port will be automatically exposed to public
        env:
          - name: "foo"
            value: "bar"
        cpu: "100m"
```

组件定义了一些通用的运行时特性。使用 cue 模板语法将必要的部分进行提取在 应用定义 时进行填写。
上面的示例就将 image port env cpu 等参数模板化到了组件定义中，在 Application 中使用时进行填写。

这样对于开发人员，仅需要关注 image port 这些 “必要” 事项，而不需要关注其他的部分。甚至不需要了解 k8s。

## 工作负载定义(WorkloadDefinition)

工作负载类型是给定组件定义的关键特征。工作负载类型由平台提供，以便用户可以检查平台并了解哪些工作负载类型可供使用。

![concepts](https://github.com/oam-dev/spec/raw/master/assets/concepts.png)

工作负载类型定义是由平台提供的，例如:

- Job Deployment StatefulSet 等由 k8s 提供的工作负载类型。
- RDS OceanBase Redis 等由云数据库提供的工作负载类型。

```yaml
kind: WorkloadDefinition
metadata:
  name: redis.cache.aliyun.com
spec:
  definitionRef:
    name: redis.cache.aliyun.com
```

在 OAM 中目前仅定义了一个 `ContainerizedWorkload`, 它是一个容器化的工作负载。
一个示例：

```yaml
apiVersion: core.oam.dev/v1alpha2
kind: ContainerizedWorkload
spec:
  osType: linux
  arch: amd64
  containers:
    - name: nginx
      image: docker.io/library/nginx:alpine
      resources:
        cpu:
          required: 1
        memory:
          required: 1G
        gpu:
          required: 1
        volumes:
          - name: busybox-volume
            mountPath: /etc/nginx/conf.d
            accessMode: RW
            sharingPolicy: Exclusive
            disk:
              required: 1G
              ephemeral: true
        extended:
          - name: ext.example.com/v1.MotionSensor
            required: "1"
          - name: ext.example.com/v2beta4.ServoModel
            required: z141155-t100
        cmd: ["/bin/sh"]
        args: ["-c", "while true; do echo 'Hello World!'; sleep 1; done"]
        env:
          - name: "ADMIN_USER"
            value: "admin"
        config:
          - path: "/etc/access/default_user.txt"
            value: "admin" # This is a literal value
          - path: "/var/run/db-data"
            fromParam: "sourceData" # This will cause the value to be read from the parameter whose name is `sourceData`
        ports:
          - name: http
            containerPort: 80
            protocol: TCP
        livenessProbe:
          httpGet:
            path: /
        readinessProbe:
          httpGet:
            path: /
        imagePullSecret: "docker-registry-secret"
```

虽然 ContainerizedWorkload 定义和 Pod 定义相似，但 ContainerizedWorkload 的目的是定义一个通用的容器化工作负载。
意味着 ContainerizedWorkload 可以用于任何容器化的工作负载，可以使用 k8s 执行，也可以使用其他平台(例如 Docker Compose,Docker Swarm）执行。

## 应用范围定义(ScopeDefinition)

应用程序范围通过提供具有共同组行为的不同形式的应用程序边界，用于将组件组合成逻辑应用程序。

应用范围具有以下一般特征：

- 当定义一组组件实例共有的行为或元数据时，应该使用应用程序范围。
- 一个组件可以同时部署到多个不同类型的应用范围中。
- 应用范围类型可以确定组件是否可以同时部署到同一应用范围类型的多个实例中。
- 应用范围可以用作组件组和基础设施提供的功能（例如网络）或外部功能（例如身份提供者）之间的连接机制。

![scopes-diagram](https://github.com/oam-dev/spec/raw/master/assets/scopes-diagram-1.png)

此示例显示了两种范围类型，其中分布有四个组件。

组件 A、B 和 C 部署到相同的运行状况范围。运行状况范围将收集有关其组成组件的聚合运行状况信息，这些信息在组件升级操作期间进行评估。
健康范围提供的查询信息可以被需要根据一组组件的总体健康状况评估和/或执行操作的特征或组件进一步使用。
这是应用程序的基本分组结构，它提供了组件之间依赖关系的松散定义。

组件 A 在其自己的网络范围内与组件 B、C 和 D 隔离。这允许基础设施运营商为不同的组件组提供不同的 SDN 设置，例如后端组件上更受限制的入站/出站规则。

```yaml
apiVersion: core.oam.dev/v1beta1
kind: ScopeDefinition
metadata:
  name: healthscopes.core.oam.dev
spec:
  allowComponentOverlap: true
  definitionRef:
    name: healthscopes.core.oam.dev
```

以 HealthScope 为例:

定义：

```yaml
apiVersion: core.oam.dev/v1alpha2
kind: HealthScope
metadata:
  name: app-health
spec:
  probe-timeout: 5
  probe-interval: 5
```

使用时：

```yaml
kind: Application
metadata:
  name: example-appconfig
spec:
  components:
    - componentName: example-component
      traits: ...
      scopes:
        healthscopes.core.oam.dev: example-health-scope
```

以 NetworkScope 为例:

网络范围将组件组合在一起并将它们链接到网络或 SDN。网络本身必须由应用运营商定义和运营。
流量管理特征也可以查询网络范围，以确定服务网格的可发现性边界或 API 网关的 API 边界。
如果未分配网络范围，则平台必须将应用程序加入默认网络。
在该默认网络中，应用程序配置中的所有组件必须能够相互通信，并且健康探测器必须能够联系任何定义健康检查规则的组件。
然而，加入的网络依赖于平台。
例如，基于集群的环境（如 Kubernetes）声明集群范围的网络。
相反，真正的无服务器实现可以将组件连接到仅包含组件和健康检查探针的网络。

```yaml
apiVersion: standard.oam.dev/v1alpha2
kind: NetworkScope
metadata:
  name: example-vpc-network
spec:
  allowComponentOverlap: true # can be skipped
  networkId: cool-vpc-network
  subnetIds:
    - cool-subnetwork
    - cooler-subnetwork
    - coolest-subnetwork
  internetGatewayType: nat
```

使用时：

```yaml
kind: Application
metadata:
  name: example-appconfig
spec:
  components:
    - componentName: example-component
      traits: ...
      scopes:
        networkscopes.standard.oam.dev: example-vpc-network
```

## 特征定义(TraitDefinition)

特征是一种任意的运行时覆盖，它为组件工作负载实例增加了操作特性。它为那些扮演应用程序操作员角色的人提供了一个机会，可以对组件的配置做出具体的决定，而不必涉及组件提供者或破坏组件封装。

特征可以是适用于单个组件的分布式应用程序的任何配置，例如流量路由规则（例如，负载平衡策略、网络入口路由、断路、速率限制）、自动扩展策略、升级策略等.

以 openvela 中的 scaler 自动扩展策略为例:

```yaml
apiVersion: core.oam.dev/v1beta1
kind: TraitDefinition
metadata:
  name: scaler
spec:
  appliesToWorkloads:
    - deployments.apps
  schematic:
    cue:
      template: |
        outputs: cpuscaler: {
          apiVersion: "autoscaling/v1"
          kind: "HorizontalPodAutoscaler"
          metadata:
            name: context.name
          spec: {
            scaleTargetRef: {
              apiVersion: parameter.targetAPIVersion
              kind: parameter.targetKind
              name: context.name
            }
            minReplicas:  parameter.min
            maxReplicas:  parameter.max
            targetCPUUtilizationPercentage: parameter.cpuUtil
          }
        }
        parameter: {
          // +usage=Specify the minimal number  of replicas to which the autoscaler can scale down
          min: *1 | int
          //  +usage=Specify the maximum number of of replicas to which the autoscaler can scale up
          max: *10 | int
          // +usage=Specify the average CPU utilization, for example, 50 means the CPU usage is 50%
          cpuUtil: *50 | int
          // +usage=Specify the apiVersion of scale target
          targetAPIVersion: *"apps/v1" | string
          // +usage=Specify the kind of scale target
          targetKind: *"Deployment" | string
        }
```

特征会根据应用程序的配置对最终的渲染输出进行修改/增添。特征是一个强大的功能，可以通过特征将运维工作与特定的应用程序分离。

## 应用定义(Application)

Application 实体定义了在部署应用程序后将被实例化的组件列表。

用户将指定每个组件的最终参数化以及用于增强其功能或改变其行为的特征。此外，可以指定一组对不同组件子集进行分组的范围。

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: my-example-app
  annotations:
    version: v1.0.0
    description: "Brief description of the application"
spec:
  components:
    - name: publicweb
      type: webservice
      properties: # properties 会被传入 ComponentDefinition 中作为参数
        image: example/web-ui:v1.0.2@sha256:verytrustworthyhash
        param_1: "enabled" # param_1 is defined on the web-ui component
      traits: # 运维特征
        - type: ingress # ingress trait providing a public endpoint for the publicweb component of the application.
          properties: #
            path: /
            port: 8080
        - type: scaler
          properties: # 参数会被传入 scaler trait 中进行渲染
            min: 1
            max: 5
            cpuUtil: 80
      scopes:
        # 上面定义的类型为 healthscopes.core.oam.dev ,名称为 app-health 的 scope 会被应用到组件上，更新 probe-timeout 和 probe-interval 为 5s 。
        "healthscopes.core.oam.dev": "app-health" #
    - name: backend
      type: company/test-backend # test-backend is referenced from other namespace
      properties:
        debug: "true" # debug is a parameter defined in the test-backend component.
      scopes:
        "healthscopes.core.oam.dev": "app-health" # An application level health scope including both components.
```

## 小结

OAM 总体上讲整个部署流程分为三个角色：

- 开发。创建并填写 Application 实体，并且指定组件的参数化和运维特征。
- 运维。根据应用的运维特征 Trait 对组件进行运维特征的渲染。
- 平台提供商。通过 WorkloadDefinition 提供不同类型的工作负载支持。

所有的工作都是围绕 Application 进行。

一个完整的流程为：

1. 平台创建 WorkloadDefinition , 表示支持 Deployment 类型工作负载。可以是非 k8s 的工作负载，例如云服务商提供的在线数据库服务。
2. 平台创建 ComponentDefinition , 引用 Deployment 类型 WorkloadDefinition，并且设置默认值并进行部分参数化，例如将镜像，启动参数进行参数化。如果对该 ComponentDefinition 进行渲染则会输出一个 Deployment。
3. 平台创建 ScopeDefinition , 表示平台提供的通用能力，而且这种能力是跨 workload 类型的。例如 网络隔离。（虽然这些也都可以使用 Trait 完成）
4. 运维人员创建 Trait . 例如 storage trait。storage trait 设置为对 Deployment 类型工作负载生效，会对 Deployment 附加 Volume。但是 storage trit 还缺少一些必要的信息，所以 Trait 上还定义着需要接受哪些参数。例如哪个目录需要使用持久化存储，而这些必要的信息应当由 开发人员提供（如果不提供则使用默认值），也就是要被定义在 Application 上。（trait 的解释执行主要使用 cue 语言完成）
5. 开发人员创建 Application 。并填写好应用有关配置，例如镜像，启动参数，运维特征等。这里的运维特征是 Trait 中需要的信息，例如需要持久化的路经，对外提供服务的端口，协议等。
6. OAM 实现将所有信息整合，并且渲染出部署资源。
7. 部署并观测。

OAM 核心可以理解为一个部署资源渲染流水线，流水线从开发人员开始，聚合多方信息，一步一步对最终部署的资源进行加工。
在这个流水线上，开发、运维、平台、都是流水线上的工人，各司其职而无需关心上下游而达到解耦的效果。
