## Elastic Cloud on Kubernetes

### 部署 ECK

1. 安装 CRD

   ```sh
   $ kubectl create -f https://download.elastic.co/downloads/eck/2.4.0/crds.yaml
   customresourcedefinition.apiextensions.k8s.io/agents.agent.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/apmservers.apm.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/beats.beat.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/elasticmapsservers.maps.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/elasticsearches.elasticsearch.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/enterprisesearches.enterprisesearch.k8s.elastic.co created
   customresourcedefinition.apiextensions.k8s.io/kibanas.kibana.k8s.elastic.co created
   ```

2. 安装带有 RBAC rules 的 operator：

   ```sh
   $ kubectl apply -f https://download.elastic.co/downloads/eck/2.4.0/operator.yaml
   namespace/elastic-system created
   serviceaccount/elastic-operator created
   secret/elastic-webhook-server-cert created
   configmap/elastic-operator created
   clusterrole.rbac.authorization.k8s.io/elastic-operator created
   clusterrole.rbac.authorization.k8s.io/elastic-operator-view created
   clusterrole.rbac.authorization.k8s.io/elastic-operator-edit created
   clusterrolebinding.rbac.authorization.k8s.io/elastic-operator created
   service/elastic-webhook-server created
   statefulset.apps/elastic-operator created
   validatingwebhookconfiguration.admissionregistration.k8s.io/elastic-webhook.k8s.elastic.co created
   ```

   > ECK operator 默认在 elastic-system 命名空间中运行。建议您为工作负载选择专用命名空间，而不是使用`elastic-system` 或 `default` 命名空间。

3. 查看 operator 日志：

   ```sh
   $ kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
   ```

### 部署 Elasticsearch 集群

#### 自定义 `elastic` 用户密码

Operator 默认会创建包含默认用户 `elastic` 密码的 secret（`elastk8s-es-elastic-user`），我们通过创建基本身份验证的 secret 为 `elastic` 用户指定密码，则 operator 不再创建包含 `elastic` 密码的 secret。

```sh
$ $ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
  namespace: elastic-system
type: kubernetes.io/basic-auth
stringData:
  username: elastic
  password: elastic.admin
  roles: superuser
EOF
```

查看 secret 详细信息：

```sh
$ kubectl get secret -n elastic-system secret-basic-auth -o yaml
apiVersion: v1
data:
  password: ZWxhc3RpYy5hZG1pbg==
  roles: c3VwZXJ1c2Vy
  username: ZWxhc3RpYw==
kind: Secret
...
```

#### 创建 Elasticsearch 集群

此示例设置一个具有 4 个节点的 Elasticsearch 集群，其中 1 个 master 节点，3 个 data 节点。

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elastk8s-cluster
  namespace: elastic-system
  annotations:
    eck.k8s.elastic.co/downward-node-labels: "topology.kubernetes.io/zone"
namespace: elastic-system
spec:
  version: 8.4.1
  auth:
    fileRealm:
    - secretName: secret-basic-auth
  # DeleteOnScaledownAndClusterDeletion(default): all pvc are deleted together with the es cluster
  # DeleteOnScaledownOnly: keeps the pvc when deleting the es cluster
  volumeClaimDeletePolicy: DeleteOnScaledownOnly
  updateStrategy:
    changeBudget:
      maxSurge: -1
      maxUnavailable: 1
  podDisruptionBudget:
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          elasticsearch.k8s.elastic.co/cluster-name: elastk8s-cluster
  nodeSets:
  # 1 dedicated master node
  - name: master-node
    count: 1
    config:
      node.roles: ["master"]
      xpack.ml.enabled: false
      node.store.allow_mmap: false
      cluster.routing.allocation.awareness.attributes: k8s_node_name,zone
      node.attr.zone: ${ZONE}
    podTemplate:
      spec:
        #initContainers:
        #- name: sysctl
        #  securityContext:
        #    privileged: true
        #    runAsUser: 0
        #  command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          resources:
            limits:
              memory: 8Gi
              cpu: 2
          env:
          - name: ZONE
            valueFrom:
              fieldRef:
                fieldPath: metadata.annotations['topology.kubernetes.io/zone']
          - name: ES_JAVA_OPTS
            value: "-Xms2g -Xmx2g"
        affinity:
          podAntiAffinity:
            #requiredDuringSchedulingIgnoredDuringExecution:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    elasticsearch.k8s.elastic.co/cluster-name: elastk8s-cluster
                topologyKey: kubernetes.io/hostname
        topologySpreadConstraints:
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
              matchLabels:
                elasticsearch.k8s.elastic.co/cluster-name: elastk8s-cluster
                elasticsearch.k8s.elastic.co/statefulset-name: elastk8s-cluster-es-default
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
        storageClassName: aisino-sc
  # 3 ingest-data nodes
  - name: data-nodes
    count: 3
    config:
      node.roles: ["data", "ingest"]
      xpack.ml.enabled: false
      node.store.allow_mmap: false
      cluster.routing.allocation.awareness.attributes: k8s_node_name,zone
      node.attr.zone: ${ZONE}
    podTemplate:
      spec:
        #initContainers:
        #- name: sysctl
        #  securityContext:
        #    privileged: true
        #    runAsUser: 0
        #  command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          resources:
            limits:
              memory: 8Gi
              cpu: 2
          env:
          - name: ZONE
            valueFrom:
              fieldRef:
                fieldPath: metadata.annotations['topology.kubernetes.io/zone']
          - name: ES_JAVA_OPTS
            value: "-Xms2g -Xmx2g"
        affinity:
          podAntiAffinity:
            #requiredDuringSchedulingIgnoredDuringExecution:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    elasticsearch.k8s.elastic.co/cluster-name: elastk8s-cluster
                topologyKey: kubernetes.io/hostname
          #nodeAffinity:
          #  requiredDuringSchedulingIgnoredDuringExecution:
          #    nodeSelectorTerms:
          #    - matchExpressions:
          #      - key: environment
          #        operator: In
          #        values:
          #        - e2e
          #        - production
          #  preferredDuringSchedulingIgnoredDuringExecution:
          #    - weight: 1
          #      preference:
          #        matchExpressions:
          #        - key: diskType
          #          operator: In
          #          values:
          #          - ssd
        # 
        topologySpreadConstraints:
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
              matchLabels:
                elasticsearch.k8s.elastic.co/cluster-name: elastk8s-cluster
                elasticsearch.k8s.elastic.co/statefulset-name: elastk8s-cluster-es-default
        #nodeSelector:
        #  diskType: ssd
        #  environment: production
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 500Gi
        storageClassName: aisino-sc
EOF
```

Operator 自动创建和管理 Kubernetes 资源，以实现 Elasticsearch 集群所需的状态。创建所有资源并准备好使用集群可能需要几分钟时间。

> 设置 node.store.allow_mmap: false 会影响性能，应该针对生产工作负载进行调整。

#### 监控集群健康和创建进度

获取 Kubernetes 集群中当前 Elasticsearch 集群的概览，包括运行状况、版本和节点数：

```sh
$ kubectl get elasticsearch -n elastic-system 
NAME               HEALTH   NODES   VERSION   PHASE   AGE
elastk8s-cluster   green    4       8.4.1     Ready   66m
```

创建集群时，没有 HEALTH 状态，PHASE 为空。一段时间后，PHASE 变为 Ready，HEALTH 变为绿色。 HEALTH 状态来自 [Elasticsearch’s cluster health API](https://www.elastic.co/guide/en/elasticsearch/reference/8.4/cluster-health.html).

查看 Pod 运行状态：

```sh
$ kubectl get pods -n elastic-system --selector='elasticsearch.k8s.elastic.co/cluster-name=elastk8s-cluster'
NAME                                READY   STATUS    RESTARTS   AGE
elastk8s-cluster-es-data-nodes-0    1/1     Running   0          68m
elastk8s-cluster-es-data-nodes-1    1/1     Running   0          68m
elastk8s-cluster-es-data-nodes-2    1/1     Running   0          68m
elastk8s-cluster-es-master-node-0   1/1     Running   0          68m
```

#### 请求 Elasticsearch 访问权限

ClusterIP 服务会自动为您的集群创建：

```sh
$ kubectl get service elastk8s-cluster-es-http -n elastic-system
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
elastk8s-cluster-es-http   ClusterIP   10.96.237.158   <none>        9200/TCP   71m
```

请求 Elasticsearch 端点

```sh
$ kubectl port-forward -n elastic-system service/elastk8s-cluster-es-http 9200
```

然后请求本地主机：

```sh
$ curl -u "elastic:elastic.admin" -k "https://localhost:9200"
{
  "name" : "elastk8s-cluster-es-data-nodes-0",
  "cluster_name" : "elastk8s-cluster",
  "cluster_uuid" : "MZhoTS78TaOLphwga9ACRw",
  "version" : {
    "number" : "8.4.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2bd229c8e56650b42e40992322a76e7914258f0c",
    "build_date" : "2022-08-26T12:11:43.232597118Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

> D不建议使用 -k 标志禁用证书验证，并且应该仅用于测试目的。检查[设置您自己的证书](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html#k8s-setting-up-your-own-certificate)。

### 部署 Kibana 实例

要部署您的 Kibana 实例，请执行以下步骤。

1. 指定一个 Kibana 实例并将其与您的 Elasticsearch 集群关联：

   ```sh
   $ cat <<EOF | kubectl apply -f -
   apiVersion: kibana.k8s.elastic.co/v1
   kind: Kibana
   metadata:
     name: elastk8s
     namespace: elastic-system
   spec:
     version: 8.4.1
     count: 1
     elasticsearchRef:
       name: elastk8s
       namespace: elastic-system
     http:
       service:
         spec:
           type: LoadBalancer # default is ClusterIP
           loadBalancerIP: 192.168.30.30
   EOF
   ```

2. 监控 Kibana 运行状况和创建进度。

   与 Elasticsearch 类似，您可以检索有关 Kibana 实例的详细信息：

   ```sh
   $ kubectl get kibana -n elastic-system 
   NAME       HEALTH   NODES   VERSION   AGE
   elastk8s   green    1       8.4.1     8m9s
   ```

   以及相关的 Pod：

   ```sh
   $ kubectl get pod -n elastic-system --selector='kibana.k8s.elastic.co/name=elastk8s'
   NAME                           READY   STATUS    RESTARTS   AGE
   elastk8s-kb-544557b7c6-h4vh4   1/1     Running   0          10m
   ```

3. 访问 Kibana。

   为 Kibana 自动创建的一个 ClusterIP 服务：

   ```sh
   $ kubectl get service elastk8s-kb-http -n elastic-system 
   NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
   elastk8s-kb-http   LoadBalancer   10.96.67.213   192.168.30.30   5601:31705/TCP   11m
   ```

   > 在浏览器中打开 https://192.168.30.30:5601

   或者使用 `kubectl port-forward` 从本地工作站访问 Kibana：

   ```sh
   $ kubectl port-forward service/elastk8s-kb-http 5601 -n elastic-system
   ```

   在浏览器中打开 https://localhost:5601。您的浏览器将显示警告，因为默认配置的自签名证书未经已知证书颁发机构验证且不受您的浏览器信任。出于本快速入门的目的，您可以暂时确认警告，但强烈建议您为任何生产部署配置有效证书。


## Filebeat

### Java 堆栈跟踪

这是一个 Java 堆栈跟踪，它提供了一个稍微复杂的示例：

```shell
Exception in thread "main" java.lang.IllegalStateException: A book has a null property
       at com.example.myproject.Author.getBookIds(Author.java:38)
       at com.example.myproject.Bootstrap.main(Bootstrap.java:14)
Caused by: java.lang.NullPointerException
       at com.example.myproject.Book.getId(Book.java:22)
       at com.example.myproject.Author.getBookIds(Author.java:35)
       ... 1 more
```

要将这些行合并为 Filebeat 中的单个事件，请对 filestream 使用以下多行配置：

```yaml
parsers:
- multiline:
    type: pattern
    pattern: '^[[:space:]]+(at|\.{3})[[:space:]]+\b|^Caused by:'
    negate: false
    match: after
```

在此示例中，模式匹配以下行：

- 以空格开头的行，后跟单词 `at` 或者 `...`
- 以单词 `Caused by: `开头的一行

### 续行

配置将任何以 `\` 字符结尾的行与后面的行合并：

```yaml
parsers:
- multiline:
    type: pattern
    pattern: '\\$'
    negate: false
    match: before
```

### Timestamps💛

来自 Elasticsearch 等服务的活动日志通常以时间戳开头，然后是有关特定活动的信息，如下例所示：

```shell
[2015-08-24 11:49:14,389][INFO ][env                      ] [Letha] using [1] data paths, mounts [[/
(/dev/disk1)]], net usable_space [34.5gb], net total_space [118.9gb], types [hfs]
```

要将这些行合并为 Filebeat 中的单个事件，请对 filestream 使用以下多行配置：

```yaml
parsers:
- multiline:
    type: pattern
    pattern: '^\[[0-9]{4}-[0-9]{2}-[0-9]{2}'
    negate: true
    match: after
```

该配置使用了 `negate:true` 和 `match:after` 设置来指定任何与指定模式不匹配的行都属于上一行。

###  Application events

有时您的应用程序日志包含以自定义标记开始和结束的事件，例如以下示例：

```shell
[2015-08-24 11:49:14,389] Start new event
[2015-08-24 11:49:14,395] Content of processing something
[2015-08-24 11:49:14,399] End event
```

要将其合并为 Filebeat 中的单个事件，请将以下多行配置与 filestream 一起使用：

```yaml
parsers:
- multiline:
    type: pattern
    pattern: 'Start new event'
    negate: true
    match: after
    flush_pattern: 'End event'
```

`flush_pattern` 选项指定将刷新当前多行的正则表达式。如果您想到指定事件开始的模式选项，`flush_pattern` 选项将指定事件的结束或最后一行。

> 如果开始/结束日志块与非多行日志混合，或者不同的开始/结束日志块相互重叠，则此示例将无法正常工作。例如，以下示例中的其他一些日志日志行将被合并到单个多行文档中，因为它们既不匹配 multiline.pattern 也不匹配 multiline.flush_pattern，并且 multiline.negate 设置为 true。
>
> ```shell
> [2015-08-24 11:49:14,389] Start new event
> [2015-08-24 11:49:14,395] Content of processing something
> [2015-08-24 11:49:14,399] End event
> [2015-08-24 11:50:14,389] Some other log
> [2015-08-24 11:50:14,395] Some other log
> [2015-08-24 11:50:14,399] Some other log
> [2015-08-24 11:51:14,389] Start new event
> [2015-08-24 11:51:14,395] Content of processing something
> [2015-08-24 11:51:14,399] End event
> ```

要将不同的配置设置应用于不同的文件，需要定义多个输入部分：

```yaml
filebeat.inputs:
- type: filestream 
  id: my-filestream-id
  paths:
    - /var/log/system.log
    - /var/log/wifi.log
- type: filestream 
  id: apache-filestream-id
  paths:
    - "/var/log/apache2/*"
  fields:
    apache: true
```

以下示例将 Filebeat 配置为忽略所有具有 gz 扩展名的文件：

```yaml
filebeat.inputs:
- type: filestream
  ...
  prospector.scanner.exclude_files: ['\.gz$']
```

> 默认情况下不排除任何文件。

以下示例将 Filebeat 配置为排除不在 /var/log 下的文件：

```yaml
filebeat.inputs:
- type: filestream
  ...
  prospector.scanner.include_files: ['^/var/log/.*']
```

> 如果是绝对路径，模式应该以 `^` 开头。

## 卸载 ECK

移除所有命名空间中的所有 Elastic 资源：

```shell
$ kubectl get namespaces --no-headers -o custom-columns=:metadata.name \
  | xargs -n1 kubectl delete elastic --all -n
```

这会删除所有底层 Elastic Stack 资源，包括它们的 Pod、Secrets、Services 等。

卸载 operator：

```shell
$ kubectl delete -f https://download.elastic.co/downloads/eck/2.4.0/operator.yaml
$ kubectl delete -f https://download.elastic.co/downloads/eck/2.4.0/crds.yaml
```
