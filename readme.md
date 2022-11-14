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



将longhorn UI发布到nodeport 30210

```bash
kubectl patch svc -n longhorn-system longhorn-frontend  -p '{"spec":{"type": "NodePort"}}'kubectl patch service longhorn-frontend --namespace=longhorn-system --type='json' --patch='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30210}]'
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
  replicas: 1
  minimumMasterNodes: 1
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
apiVersion: rbac.authorization.k8s.io/v1
```

```bash
nano filebeat/templates/clusterrolebinding.yaml 
```

```
apiVersion: rbac.authorization.k8s.io/v1
```



安装filebeat

```bash
helm install fb --namespace=efk ./filebeat
```



安装kibana

```bash
helm install kb --namespace=efk ./kibana
```



查看pod和svc

```bash
kubectl get pod -n efk
```

```bash
kubectl get svc -n efk
```



为kibana服务开放nodeport 30140端口

```bash
kubectl patch svc -n efk kb-kibana  -p '{"spec":{"type": "NodePort"}}'
kubectl patch service kb-kibana --namespace=efk --type='json' --patch='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":30140}]'
```

使用30140端口打开kibana界面进行初始化



## Filebeat注入实验

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
      volumes:
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



尝试查看pod的日志

```bash
kubectl logs app-68bc4f46f4-8gqzp
```

  内容无



对比查看另外一个pod

```bash
kubectl logs mysql-d869bcc87-45q9p
```

  很丰富



给上述app注入filebeat

```bash
nano app-filebeat.yaml
```

```
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
        - name: filebeat                        
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
            - name: logpath
              mountPath: /data/log/app/
            - name: filebeatconf
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
        - name: logpath
          emptyDir: {}
        - name: filebeatconf
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
      fields:
        pod_name: '${podName}'
        pod_ip: '${podIp}'
        pod_deploy_name: '${podDeployName}'
        pod_namespace: '${podNamespace}'
    output.elasticsearch:
      hosts: ["10.106.144.108:9200"] 
      topic: "filebeat-sidecar"
      codec.json:
        pretty: false
      keep_alive: 30s
```



启用

```bash
kubectl apply -f filebeat-cm.yaml -f app-filebeat.yaml
```



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

![image-20221114163643464](D:\github\k8s-monitoring\assets\image-20221114163643464.png)

观察loki的cluster-ip



在grafana里增加loki数据源，

[]: http://loki:3100

(如果loki安装在grafana之外的名称空间，则使用上述观察到的loki cluter-ip和端口）



其后观察使用以下范例观察收集的日志信息

```yaml
{namespace="kube-system"}
```

```yaml
{namespace="kube-system", pod=~"calico.*"}
```

```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg"
```

```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg" | logfmt | longest > 16ms
```

```yaml
{namespace="kube-system", pod=~"calico.*"} |~ "avg" | logfmt | longest > 16ms and avg >6ms
```

