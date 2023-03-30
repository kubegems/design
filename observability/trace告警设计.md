# Trace告警设计

|                            | 通过`span metrics`指标告警 | 通过`alertmanagerexporter`实现告警                  | 通过`trace controller`实现告警                          |
| -------------------------- | -------------------------- | --------------------------------------------------- | ------------------------------------------------------- |
| 现有支持情况               | 已支持                     | 需要为`opentelemetry`开发`alertmanagerexporter`组件 | 需要在`kubegems-controller`中添加`trace controller`模块 |
| 开发工作量                 | 低                         | 高                                                  | 高                                                      |
| 告警消息是否支持`trace id` | 否                         | 是                                                  | 是                                                      |
| 告警源                     | prometheus                 | opentelemetry                                       | trace controller                                        |
| 告警及时性                 | 低                         | 低                                                  | 低                                                      |
| 告警精度                   | 低                         | 高                                                  | 高                                                      |
| 查询压力                   | 低                         | 低                                                  | 高                                                      |
| 可扩展性                   | 低                         | 高                                                  | 高                                                      |
## 通过`span metrics`指标告警
### 架构
![](https://www.plantuml.com/plantuml/png/hP91Q-Cm48Nl-XM3SrxOSzWDONTX3oqKUYluCdOcZa2MdD4eYVzzHqIu2dLllMg-zzxuZVTHa2MDpc2IuDFO4Hli9J2aI6owS8FcPI8lv6WauRc1pRCTJlB79JAJYcrl9_G2J3tPYshEYH91rCSii7qiYlzCv1ya4mT5JgHHjocm4CFcZPmBLqF-01fk3YZOOQIAVR10HomZUXmqsUITuSTksMKBCHi9Oq_HbDTCrEjjGGqZ5GfQBm7Ao5IOgZulN18MVW4K_uhl6ITQLUpgpr6KBkWI2W5gHEjpqfs8fWQpCCKig0ZePkY3FzeXCPKAxYFWv2YQwkBG_3e6BygxmByLt_riykdXANKqq1YDEWa71pZPZvECSpoxvLAb-uniI_9AgauLkajrNxoKtbk_GgtNBN0ALv2WGZiCnCM_coi-sy3Vrt_5wQ7NPRlbVyZrSaX2QjBwcYsaEdr3utjbq-ZU0G00)

### 核心逻辑
此方案中我们通过spanmetrics将trace数据转换成metrics指标存至prometheus，指标如下：
- calls_total：统计span数量
- latency_bucket: 统计span耗时

### 配置流程
按照普通监控告警的形式来配置trace告警。

如我们可以使用如下的promql来获取 **过去一分钟namespace otel-demo中 error span的数量**
```
sum by(namespace, node, operation, pod, service_name, http_status_code, status_code) (increase(calls_total{namespace="otel-demo",status_code="STATUS_CODE_ERROR"}[1m]))
```

此时如果我们收到告警，想要知道是哪些trace/span出了问题，需要以`error=true`为条件查询jaeger，或者我们可以在告警消息中添加jaeger url，点击直接跳转，eg:
<http://10.12.32.41:16686/search?end=1678931784787000&limit=20&lookback=1h&maxDuration&minDuration&service=otel-demo-server&start=1678928184787000&tags=%7B%22error%22%3A%22true%22%7D>

## 为Opentelemetry开发`alertmanagerexporter`实现告警
### 架构
![](https://www.plantuml.com/plantuml/png/TLDDZzCm4BtxLupA6HLnh62r0YGE8EqAk23ruYQlZONZoOx7_VZts7KAJPIzZU8ty-QzDtcCQaLJxurDoY7rEuYnZR9GYhdSyPbMJmE2mgE7ocL5RxwxGwUVV88PShDht623ag21E-QUbuG4sZkla46uGOmiOzTN0SAapNg8zZRO0mJdWKMHmR_XFPzCO0Ln7heCgpEVNVXZJg00jAHCBOxmF91sBfg6-u43Cf4BnFT2o22qL1YkHLRwwbx653FexSC_aHkAfKd5PdccVgs8cUuDZUM8fQf9ok8RQZZitO5EdMkw2jW65maYBDH1O6RZCiEY5Piih_6f1MKCphMiZ1F2U9LwcSmWExZDDZm__Vn5wrolZ-_Nr_TebgZcAun3QmSfJnudv2dcCdd4aiKstEEg4TkWV3_Dt5qbhJxiEIWrx5a-2jg7TwlhUNvBEnpGbwY32liYnxHMxSv6K7-9Bxu4dA6t50lYvdqrgVBLvKn-pQ3QcSNP8sxHp55BSKSeTNmgHcZMVHXlT-7-wW9bq9Ab_tyeig4buQJ8oItPHfz_V9b4LUQbE8zcTwDCWdP1zsF6fjx_1G00)

### 核心逻辑
此方案需要我们为opentelemetry开发组件`alertmanagerexporter`来实现以下功能：

1. 将来自[filterprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md)的trace转换为alertmanager格式的告警消息，注意将trace的`tag`、`log`转换为alert中的`labels`和`annotations`，注意带上`trace_id`
2. 将转换后的alert直接发送至alertmanager(调用alertmanager的`POST /api/v2/alerts`接口)

注意：
1. `alertmanagerexporter`只需要配置目标`alertmanager`的地址，它本身不做任何过滤，过滤交给`filterprocessor`
2. 我们不在`alertmanagerexporter`处做聚合，它收到trace直接进行转发，聚合操作交给`alertmanager`的`group`来做


### 配置流程
在用户提交trace(trace中需要按Kubegems平台上的配置有namespace、node、pod这些信息)告警规则配置后，后端做的操作如下:

1. 将告警规则存至mysql
2. 修改该集群中`opentelemetry-collector`中`filterprocessor`的配置，即配置告警触发条件，eg:
```yaml
processors:
  filter/spans:
    spans:
      include:
        match_type: strict
        services:
          - app_1
        span_names: 
          - "GET /user"
        attributes:
          - key: namespace
            value: mynamespace
         - key: error
            value: STATUS_CODE_ERROR 
exporters:
  # to be implemented
  alertmanager:
    endpoint: kube-prometheus-stack-alertmanager.observability:9093
```
3. 创建/更新该集群的`alertmanagerconfig`资源，即配置告警接收器、路由方式等
```yaml
    routes:
    - continue: true
      matchers:
      - name: namespace
        value: mynamespace
      - name: trace_service_name
        value: app_1
      - name: trace_span_name
        value: "GET /user"
      - name: attributes_error
        value: STATUS_CODE_ERROR
      receiver: myreceiver
      repeatInterval: 1m
```

由于使用的`alertmanagerconfig`，完成这些配置后，等待告警触发即可。配置告警黑名单等操作也和现有的逻辑一致。

## 开发`tracerule.tracing.kubegems.io`的CRD和对应的controller实现告警
### 架构
![](https://www.plantuml.com/plantuml/png/TP91ZzD038Nl_XLMkR8gueGgMAq02GcGTWLSaFRYfjvssCbCwl5aj_yUfvE497Hlq_Yp_PvVRvAIQAuzKALHC2SMsCPdNDqs79GzrwnoMk6hxsv_q8y-CpJMuIhNK50KhjYrrdFCd1aRYHMd52NDg3_4UnP-RgAeiHnsQ3kWBFfwBWuRVhFtyUb2TnMzvquZx4XfIucHFIMZ4fDKX_dKRtbhJNM2AWQLw565Akv_U9P-snTX3X2YCaPpyG9rvy8ZS4i-au5a1KNOdWejsJD--l7vDUY10oQJKP24q3egxve2cGUanctWxlRdBrpRSzs-NHVqBBwJCUgrylJce-Vwb8v-z3ezqtGH06MD8TTRUtUPujNrcDNcBBAeJp1yDM8ItsPuznZ2lvgHYqJd4-SrenS9pd7KsDTXWLddF6BhevQ7DSkihlx7ZvcbZmcNUKrFXAlt3pO8RNMK3yAxwpUhyMnTPbO93svV9fHBOpSeWLKDAh5CzTs72zH2il6iU8XFgD5gRc_kXpuNyF5TkWzBsSEeQTfbpXWasMqbPkqysxW9M5oKCUFduh9pkvdzUsrpuRCTv8Bp6sDpxVy2)

### 核心逻辑
此方案需要在`kubegems-controller`中开发针对`tracerule.tracing.kubegems.io`CRD的controller，一个`tracerule`的资源大概长这样:
```yaml
apiVersion: tracing.kubegems.io/v1beta1
kind: TraceRule
metadata:
  name: test-trace-alert
  namespace: mynamespace
spec:
  groups:
  - name: test-trace-alert
    rules:
    - alert: test-trace-alert
      annotations:
        message: 'test-trace-alert: [namespace:mynamespace] [service:app_1] [span: GET /user] error count trigger alert in last 1m, current {{ $value }}'
      # evaluate time
      for: 0s
      labels:
        gems_alertname: test-trace-alert
        gems_namespace: mynamespace
        severity: error
      # config span rule
      span:
        # target service name
        service: app_1
        # target span name
        name: "GET /user"
        # time range
        duration: 1m
        # match tags in span
        matchTags:
          - type: equal
            key: error
            value: STATUS_CODE_ERROR
        # match logs in span
        matchLogs:
          - type: regex
            content: .*(error|Error).*
        # trigger value
        value: 1            
```

1. `trace controller`需要支持配置`evaluation_interval`，类似prometheus的评估间隔,eg:
```yaml
- commands: ["kubegems"]
  args: 
    - controller
    - --trace-evaluation_interval=30s  
```
2. 按照controller配置的`evaluation_interval`（评估间隔，类似promethues）,定期、并发地遍历并评估当前集群的每条`tracerule`CRD，并对条`tracerule`做以下事情：
   - 按照`tracerule`的`span`、`duration`(查询的时间范围)配置，在`elasticsearch`中查询trace，并综合`for`(评估时间)、`value`（触发值）等配置，判断该条`tracerule`是否应该触发告警
   - 如果需要触发告警，则将trace信息转换为alertmanager接收的格式，所有`trace tags`都转换为`alert labels`，并将`trace_id`也加入`alert labels`,方便alertmanager按照`trace_id`做告警分组
   - 发送告警至alertmanager
3. 之后的逻辑和现有的告警逻辑一致

### 配置流程
在用户提交trace(trace中需要按Kubegems平台上的配置有namespace、node、pod这些信息)告警规则配置后，后端做的操作如下:

1. 将告警规则存至mysql
2. 创建/更新集群中`tracerule`资源，即配置告警触发条件，eg:
3. 创建/更新该集群的`alertmanagerconfig`资源，即配置告警接收器、路由方式等
```yaml
    routes:
    - continue: true
      matchers:
      - name: namespace
        value: mynamespace
      - name: trace_service_name
        value: app_1
      - name: trace_span_name
        value: "GET /user"
      - name: attributes_error
        value: STATUS_CODE_ERROR
      receiver: myreceiver
      repeatInterval: 1m
```

由于使用的`alertmanagerconfig`，完成这些配置后，等待告警触发即可。配置告警黑名单等操作也和现有的逻辑一致。

