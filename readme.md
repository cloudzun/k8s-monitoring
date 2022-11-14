# 事件收集日志分析





## 实现Elasticsearch+filebeat+Kibana Stack



### 部署块存储方案 

安装longhorn

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.2.4/deploy/longhorn.yaml
```



将longhorn设置为默认存储类

````bash
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
````



确认存储类型

```bash
kubectl get sc
```



```bash
root@node1:~# kubectl get sc
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   4h13m

```



检查longhorn安装情况

```bash
kubectl get pod -n longhorn-system
```

```bash
root@node1:~# kubectl get pod -n longhorn-system
NAME                                        READY   STATUS    RESTARTS       AGE
csi-attacher-6454556647-cw4jc               1/1     Running   0              4h8m
csi-attacher-6454556647-gvjk8               1/1     Running   0              4h8m
csi-attacher-6454556647-vc7ws               1/1     Running   0              4h8m
csi-provisioner-869bdc4b79-r5v8j            1/1     Running   0              4h8m
csi-provisioner-869bdc4b79-rn9zp            1/1     Running   0              4h8m
csi-provisioner-869bdc4b79-v7kc9            1/1     Running   0              4h8m
csi-resizer-6d8cf5f99f-69kdl                1/1     Running   0              4h8m
csi-resizer-6d8cf5f99f-vwn4c                1/1     Running   0              4h8m
csi-resizer-6d8cf5f99f-xlcws                1/1     Running   0              4h8m
csi-snapshotter-588457fcdf-77h4k            1/1     Running   0              4h8m
csi-snapshotter-588457fcdf-fpkjf            1/1     Running   0              4h8m
csi-snapshotter-588457fcdf-jxdrs            1/1     Running   0              4h8m
engine-image-ei-4dbdb778-58sx7              1/1     Running   0              4h9m
engine-image-ei-4dbdb778-9c6mr              1/1     Running   0              4h9m
engine-image-ei-4dbdb778-csgdl              1/1     Running   0              4h9m
instance-manager-e-46afa502                 1/1     Running   0              4h9m
instance-manager-e-d020b8c8                 1/1     Running   0              4h9m
instance-manager-e-f8a6153d                 1/1     Running   0              4h9m
instance-manager-r-263a334e                 1/1     Running   0              4h8m
instance-manager-r-85b971e6                 1/1     Running   0              4h8m
instance-manager-r-a4c3d8b4                 1/1     Running   0              4h9m
longhorn-csi-plugin-hv8qf                   2/2     Running   0              4h8m
longhorn-csi-plugin-jm7kv                   2/2     Running   0              4h8m
longhorn-csi-plugin-t9z4k                   2/2     Running   0              4h8m
longhorn-driver-deployer-7c85dc8c69-wlsf6   1/1     Running   0              4h9m
longhorn-manager-h88kx                      1/1     Running   1 (4h9m ago)   4h9m
longhorn-manager-zf945                      1/1     Running   0              4h9m
longhorn-manager-zwjpw                      1/1     Running   0              4h9m
longhorn-ui-6dcd69998-9rscr                 1/1     Running   0              4h9m
```

```bash
kubectl get svc -n longhorn-system
```

```
root@node1:~# kubectl get svc -n longhorn-system
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
csi-attacher               ClusterIP   10.105.33.144    <none>        12345/TCP      4h9m
csi-provisioner            ClusterIP   10.106.35.240    <none>        12345/TCP      4h9m
csi-resizer                ClusterIP   10.101.232.11    <none>        12345/TCP      4h9m
csi-snapshotter            ClusterIP   10.107.169.238   <none>        12345/TCP      4h9m
longhorn-backend           ClusterIP   10.100.136.167   <none>        9500/TCP       4h11m
longhorn-engine-manager    ClusterIP   None             <none>        <none>         4h11m
longhorn-frontend          NodePort    10.97.31.129     <none>        80:30210/TCP   4h11m
longhorn-replica-manager   ClusterIP   None             <none>        <none>         4h11m
```



将longhorn UI发布到nodeport 30210

```bash
kubectl patch svc -n longhorn-system longhorn-frontend  -p '{"spec":{"type": "NodePort"}}'
kubectl patch service longhorn-frontend --namespace=longhorn-system --type='json' --patch='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30210}]'
```



### 部署EFK Stack

下载Helm仓库源码

```bash
wget -O helm-charts.tgz  https://github.com/elastic/helm-charts/archive/7.9.2.tar.gz
tar -zxvf helm-charts.tgz
cd helm-charts-7.9.2
```



国内替代方案

```bash
wget -O helm-charts.tgz  https://github.com/elastic/helm-charts/archive/7.9.2.tar.gz
tar -zxvf helm-charts.tgz
cd helm-charts-7.9.2
```



更改elasticsearch副本数，此处选用最低副本数

```bash
nano elasticsearch/values.yaml
```

```yaml
# Elasticsearch roles that will be applied to this nodeGroup
# These will be set as environment variables. E.g. node.master=true
roles:
  master: "true"
  ingest: "true"
  data: "true"

replicas: 1 # 将副本数量设置为1,下一行相同
minimumMasterNodes: 1

esMajorVersion: ""

# Allows you to add any config files in /usr/share/elasticsearch/config/
# such as elasticsearch.yml and log4j2.properties
esConfig: {}
#  elasticsearch.yml: |
#    key:
#      nestedkey: value
#  log4j2.properties: |
#    key = value
```



安装elasticsearch

```bash
helm install es --namespace=efk ./elasticsearch
```



修改 filebeat api version

```bash
nano filebeat/templates/clusterrole.yaml
```



```yaml
{{- if .Values.managedServiceAccount }}
apiVersion: rbac.authorization.k8s.io/v1 # 将此处版本修改为当前设置
kind: ClusterRole
metadata:
  name: {{ template "filebeat.serviceAccount" . }}-cluster-role
  labels:
    app: "{{ template "filebeat.fullname" . }}"
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    heritage: {{ .Release.Service | quote }}
    release: {{ .Release.Name | quote }}
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  - nodes
  - pods
  verbs:
  - get
  - list
  - watch
{{- end -}}
```



```bash
nano filebeat/templates/clusterrolebinding.yaml 
```



```yaml
{{- if .Values.managedServiceAccount }}
apiVersion: rbac.authorization.k8s.io/v1 # 将此处版本修改为当前设置
kind: ClusterRoleBinding
metadata:
  name: {{ template "filebeat.serviceAccount" . }}-cluster-role-binding
  labels:
    app: "{{ template "filebeat.fullname" . }}"
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    heritage: {{ .Release.Service | quote }}
    release: {{ .Release.Name | quote }}
roleRef:
  kind: ClusterRole
  name: {{ template "filebeat.serviceAccount" . }}-cluster-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: {{ template "filebeat.serviceAccount" . }}
  namespace: {{ .Release.Namespace }}
{{- end -}}
```



安装filebeat

```bash
helm install fb --namespace=efk ./filebeat
```



安装kibana

```bash
helm install kb --namespace=efk ./kibana
```



查看elasticsearch所对应的pv和pvc

```bash
kubectl get pvc -n efk
```



```
root@node1:~# kubectl get pvc -n efk
NAME                                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-master-elasticsearch-master-0   Bound    pvc-d6243ec6-f4cf-4ea5-9286-60a335cc2518   30Gi       RWO            longhorn       4h6m
```



```bash
kubectl get pv
```



```
root@node1:~# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS   REASON   AGE
pvc-d6243ec6-f4cf-4ea5-9286-60a335cc2518   30Gi       RWO            Delete           Bound    efk/elasticsearch-master-elasticsearch-master-0   longhorn                4h6m

```



通过longhorn页面查看pv和pvc信息

![image-20221114173727412](readme.assets/image-20221114173727412.png)



查看pod和svc

```bash
kubectl get pod -n efk
```



```
root@node1:~/helm-charts-7.9.2# kubectl get pod -n efk
NAME                         READY   STATUS    RESTARTS   AGE
elasticsearch-master-0       1/1     Running   0          3h55m
fb-filebeat-2t78k            1/1     Running   0          3h52m
fb-filebeat-l5chq            1/1     Running   0          3h52m
fb-filebeat-pjsd6            1/1     Running   0          3h52m
kb-kibana-76c88c8965-lfdm5   1/1     Running   0          3h51m
```



```bash
kubectl get svc -n efk
```



```
root@node1:~/helm-charts-7.9.2# kubectl get svc -n efk
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch-master            ClusterIP   10.101.66.117    <none>        9200/TCP,9300/TCP   3h56m
elasticsearch-master-headless   ClusterIP   None             <none>        9200/TCP,9300/TCP   3h56m
kb-kibana                       NodePort    10.103.185.165   <none>        5601:30140/TCP      3h52m
```



为kibana服务开放nodeport 30140端口

```bash
kubectl patch svc -n efk kb-kibana  -p '{"spec":{"type": "NodePort"}}'
kubectl patch service kb-kibana --namespace=efk --type='json' --patch='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30140}]'
```



使用30140端口打开kibana界面进行初始化

![image-20221114173842879](readme.assets/image-20221114173842879.png)



![image-20221114173928151](readme.assets/image-20221114173928151.png)



![image-20221114174022061](readme.assets/image-20221114174022061.png)

![image-20221114174051346](readme.assets/image-20221114174051346.png)



![image-20221114174159612](readme.assets/image-20221114174159612.png)



## Filebeat注入实验

查看命名空间 efk 中的 pod

```
 kubectl get pod -n efk
```



```
root@node1:~# kubectl get pod -n efk
NAME                         READY   STATUS    RESTARTS   AGE
elasticsearch-master-0       1/1     Running   0          4h23m
fb-filebeat-2t78k            1/1     Running   0          4h20m
fb-filebeat-l5chq            1/1     Running   0          4h20m
fb-filebeat-pjsd6            1/1     Running   0          4h20m
kb-kibana-76c88c8965-lfdm5   1/1     Running   0          4h18m
```



查看其中某个pod的日志

```
kubectl logs fb-filebeat-pjsd6  -n efk
```



```
2022-11-14T09:52:41.662Z        INFO    [monitoring]    log/log.go:145  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":30140,"time":{"ms":68}},"total":{"ticks":68100,"time":{"ms":93},"value":68100},"user":{"ticks":37960,"time":{"ms":25}}},"handles":{"limit":{"hard":1048576,"soft":1048576},"open":20},"info":{"ephemeral_id":"6e1622b9-df49-421e-b048-0913fba358bd","uptime":{"ms":15660093}},"memstats":{"gc_next":25772688,"memory_alloc":14153344,"memory_total":4278960264},"runtime":{"goroutines":75}},"filebeat":{"events":{"added":68,"done":68},"harvester":{"closed":1,"files":{"394097ab-8b7c-4191-bf7e-bbc1fe38991b":{"last_event_published_time":"2022-11-14T09:52:18.596Z","last_event_timestamp":"2022-11-14T09:52:18.010Z","read_offset":2488,"size":2488},"3a32633d-f952-4acd-a76e-bf04e7e5c00a":{"last_event_published_time":"2022-11-14T09:52:28.958Z","last_event_timestamp":"2022-11-14T09:52:28.622Z","read_offset":2425,"size":1816},"52ba1ebd-1ab6-43ec-b84a-98a55936fbb6":{"last_event_published_time":"2022-11-14T09:52:23.058Z","last_event_timestamp":"2022-11-14T09:52:16.802Z","read_offset":1515},"62895648-a424-412c-94eb-2d5a226d0efd":{"last_event_published_time":"2022-11-14T09:52:33.571Z","last_event_timestamp":"2022-11-14T09:52:32.247Z","read_offset":8606,"size":11561},"f00deb2a-beba-447b-bfbb-7aff831ff0e6":{"last_event_published_time":"2022-11-14T09:52:23.642Z","last_event_timestamp":"2022-11-14T09:52:16.562Z","name":"/var/log/containers/hello-27806992-59lcm_default_hello-506e2399a9dc6ab04ff30e2dcd73954a1cee6da5ab6f5b03f9a66fd9beb1de74.log","read_offset":203,"size":203,"start_time":"2022-11-14T09:52:23.641Z"},"f453639b-a0f1-4cb3-97ea-3cd77775712a":{"last_event_published_time":"2022-11-14T09:52:23.595Z","last_event_timestamp":"2022-11-14T09:52:20.264Z","read_offset":3049}},"open_files":8,"running":8,"started":1}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":65,"batches":5,"total":65}},"pipeline":{"clients":1,"events":{"active":0,"filtered":3,"published":65,"total":68},"queue":{"acked":65}}},"registrar":{"states":{"cleanup":1,"current":39,"update":68},"writes":{"success":6,"total":6}},"system":{"load":{"1":0.57,"15":0.94,"5":1.03,"norm":{"1":0.1425,"15":0.235,"5":0.2575}}}}}}
```



可以到kibana界面中定位相关日志信息

![image-20221114175647616](readme.assets/image-20221114175647616.png)





创建非云原生应用实例

```bash
nano app.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
    env: release
spec:
  selector:
    matchLabels:
      app: app
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  # minReadySeconds: 30
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: registry.cn-beijing.aliyuncs.com/dotbalo/alpine:3.6 
          imagePullPolicy: IfNotPresent
          volumeMounts:
          - name: logpath
            mountPath: /opt/
          env:
            - name: TZ
              value: "Asia/Shanghai"
            - name: LANG
              value: C.UTF-8
            - name: LC_ALL
              value: C.UTF-8
          command:
            - sh
            - -c
            - while true; do date >> /opt/date.log; sleep 2;  done  
      volumes: #在pod volume上存放日志
        - name: logpath
          emptyDir: {}
```

```bash
kubectl apply -f app.yaml 
```



查看pod

```bash
kubectl get pod
```



```
root@node1:~# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
app-68bc4f46f4-cfvr4    1/1     Running   0          5s
mysql-d869bcc87-45q9p   1/1     Running   0          7h39m
```



尝试查看pod的日志

```bash
kubectl logs app-68bc4f46f4-cfvr4
```

```
root@node1:~# kubectl logs app-68bc4f46f4-cfvr4
root@node1:~#
```

​		内容无



对比查看另外一个pod

```bash
kubectl logs mysql-d869bcc87-45q9p
```

```
2022-11-14T02:20:10.126154Z 0 [Warning] 'user' entry 'root@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.126218Z 0 [Warning] 'user' entry 'mysql.session@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.126231Z 0 [Warning] 'user' entry 'mysql.sys@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.126252Z 0 [Warning] 'db' entry 'performance_schema mysql.session@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.126256Z 0 [Warning] 'db' entry 'sys mysql.sys@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.126263Z 0 [Warning] 'proxies_priv' entry '@ root@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.128150Z 0 [Warning] 'tables_priv' entry 'user mysql.session@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.128197Z 0 [Warning] 'tables_priv' entry 'sys_config mysql.sys@localhost' ignored in --skip-name-resolve mode.
2022-11-14T02:20:10.131848Z 0 [Note] Event Scheduler: Loaded 0 events
2022-11-14T02:20:10.132107Z 0 [Note] mysqld: ready for connections.
Version: '5.7.23'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
2022-11-14T03:52:20.912938Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 332284ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
```

​		很丰富



给上述app注入filebeat

```bash
nano app-filebeat.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
    env: release
spec:
  selector:
    matchLabels:
      app: app
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  # minReadySeconds: 30
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: filebeat   # sidecar容器                     
          image: registry.cn-beijing.aliyuncs.com/dotbalo/filebeat:7.10.2 
          resources:
            requests:
              memory: "100Mi"
              cpu: "10m"
            limits:
              cpu: "200m"
              memory: "300Mi"
          imagePullPolicy: IfNotPresent
          env:
            - name: podIp
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
            - name: podName
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: podNamespace
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: podDeployName
              value: app
            - name: TZ
              value: "Asia/Shanghai"
          securityContext:
            runAsUser: 0
          volumeMounts:
            - name: logpath # 指向穿针引线的emptyDir
              mountPath: /data/log/app/
            - name: filebeatconf # 指向配置文件
              mountPath: /usr/share/filebeat/filebeat.yml 
              subPath: usr/share/filebeat/filebeat.yml
        - name: app
          image: registry.cn-beijing.aliyuncs.com/dotbalo/alpine:3.6 
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: logpath
              mountPath: /opt/
          env:
            - name: TZ
              value: "Asia/Shanghai"
            - name: LANG
              value: C.UTF-8
            - name: LC_ALL
              value: C.UTF-8
          command:
            - sh
            - -c
            - while true; do date >> /opt/date.log; sleep 2;  done 
      volumes:
        - name: logpath #沟通sidecar容器和app容器的桥梁作用显现
          emptyDir: {}
        - name: filebeatconf #定义日志字段和发送目标的配置文件
          configMap:
            name: filebeatconf
            items:
              - key: filebeat.yml
                path: usr/share/filebeat/filebeat.yml
```



创建filebeat配置文件

```bash
nano  filebeat-cm.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeatconf
data:
  filebeat.yml: |-
    filebeat.inputs:
    - input_type: log
      paths:
        - /data/log/*/*.log
      tail_files: true
      fields: # 日志字段
        pod_name: '${podName}'
        pod_ip: '${podIp}'
        pod_deploy_name: '${podDeployName}'
        pod_namespace: '${podNamespace}'
    output.elasticsearch:
      hosts: ["10.106.144.108:9200"] # 此处需要填充elasticsearch服务器地址
      topic: "filebeat-sidecar"
      codec.json:
        pretty: false
      keep_alive: 30s
```



```bash
kubectl apply -f filebeat-cm.yaml -f app-filebeat.yaml
```



```bash
root@node1:~# kubectl apply -f filebeat-cm.yaml -f app-filebeat.yaml
configmap/filebeatconf unchanged
deployment.apps/app configured
root@node1:~# kubectl get pod
NAME                    READY   STATUS        RESTARTS   AGE
app-68bc4f46f4-cfvr4    1/1     Terminating   0          7m46s
app-6d8c4bc6db-mtbj7    2/2     Running       0          5s
mysql-d869bcc87-45q9p   1/1     Running       0          7h47m	
```



查看pod构造



```bash
Containers:
  filebeat:
    Container ID:   docker://1f32f7d90ef53c04f932e71feb7009a4f5e5ac6b1e3bde97ac6de35ce2c215e2
    Image:          registry.cn-beijing.aliyuncs.com/dotbalo/filebeat:7.10.2
    Image ID:       docker-pullable://registry.cn-beijing.aliyuncs.com/dotbalo/filebeat@sha256:0b124b6362544e0a9b9636a187d7487c82b2865f1c3ce915b6d5dd1229ffb4a7
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 14 Nov 2022 18:07:16 +0800
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     200m
      memory:  300Mi
    Requests:
      cpu:     10m
      memory:  100Mi
    Environment:
      podIp:           (v1:status.podIP)
      podName:        app-6d8c4bc6db-mtbj7 (v1:metadata.name)
      podNamespace:   default (v1:metadata.namespace)
      podDeployName:  app
      TZ:             Asia/Shanghai
    Mounts:
      /data/log/app/ from logpath (rw)
      /usr/share/filebeat/filebeat.yml from filebeatconf (rw,path="usr/share/filebeat/filebeat.yml")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7xmvt (ro)
  app:
    Container ID:  docker://b0b04fef0d22866353271751a3c350d132b40d1e6c87e36b108739674523f5fa
    Image:         registry.cn-beijing.aliyuncs.com/dotbalo/alpine:3.6
    Image ID:      docker-pullable://registry.cn-beijing.aliyuncs.com/dotbalo/alpine@sha256:36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      while true; do echo cloudzun is here && date >> /opt/date.log; sleep 2;  done
    State:          Running
      Started:      Mon, 14 Nov 2022 18:07:17 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      TZ:      Asia/Shanghai
      LANG:    C.UTF-8
      LC_ALL:  C.UTF-8
    Mounts:
      /opt/ from logpath (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7xmvt (ro)

```

​		 可以观察到filebeat作为sidecare容器和app容器并置在一个pod



```bash
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  49s   default-scheduler  Successfully assigned default/app-6d8c4bc6db-mtbj7 to node1
  Normal  Pulled     48s   kubelet            Container image "registry.cn-beijing.aliyuncs.com/dotbalo/filebeat:7.10.2" already present on machine
  Normal  Created    48s   kubelet            Created container filebeat
  Normal  Started    48s   kubelet            Started container filebeat
  Normal  Pulled     47s   kubelet            Container image "registry.cn-beijing.aliyuncs.com/dotbalo/alpine:3.6" already present on machine
  Normal  Created    47s   kubelet            Created container app
  Normal  Started    47s   kubelet            Started container app
```



到kibana查看pod日志

![image-20221114182151624](readme.assets/image-20221114182151624.png)



![image-20221114182229268](readme.assets/image-20221114182229268.png)



![image-20221114182308087](readme.assets/image-20221114182308087.png)



## 使用Loki构建轻量级日志收集体系



使用helm安装loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack -n monitoring
```



检查安装情况

```bash
kubectl get pod -n monitoring | grep loki
kubectl get svc -n monitoring | grep loki
```



```bash
root@node1:~# kubectl get pod -n monitoring | grep loki
loki-0                                 1/1     Running   0          6h11m
loki-promtail-22nfd                    1/1     Running   0          6h11m
loki-promtail-6f5wl                    1/1     Running   0          6h11m
loki-promtail-957s4                    1/1     Running   0          6h11m
root@node1:~# kubectl get svc -n monitoring | grep loki
loki                    ClusterIP   10.107.108.73    <none>        3100/TCP                        6h11m
loki-headless           ClusterIP   None             <none>        3100/TCP                        6h11m
loki-memberlist         ClusterIP   None             <none>        7946/TCP                        6h11m
```



观察loki的cluster-ip



在grafana里增加loki数据源，

![image-20221114165928698](readme.assets/image-20221114165928698.png)

​	注意：如果loki安装在grafana之外的名称空间，则使用上述观察到的loki cluter-ip和端口）

![image-20221114170348619](readme.assets/image-20221114170348619.png)



其后观察使用以下范例观察收集的日志信息

```yaml
{namespace="kube-system"}
```

![image-20221114170447199](readme.assets/image-20221114170447199.png)



```yaml
{namespace="kube-system", pod=~"calico.*"}
```

![image-20221114170628964](readme.assets/image-20221114170628964.png)



![image-20221114170709099](readme.assets/image-20221114170709099.png)



```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg"
```

![image-20221114170754240](readme.assets/image-20221114170754240.png)



```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg" | logfmt | longest > 16ms
```

![image-20221114171008923](readme.assets/image-20221114171008923.png)



```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg" | logfmt | longest > 16ms and avg >6ms
```

![image-20221114171111960](readme.assets/image-20221114171111960.png)



亦可在此处查看注入filebeat sidecare的pod日志

```yaml
{namespace="default", pod=~"app-6d8c4bc6db-mtbj7"}
```

![image-20221114182625091](readme.assets/image-20221114182625091.png)



![image-20221114182726583](readme.assets/image-20221114182726583.png)