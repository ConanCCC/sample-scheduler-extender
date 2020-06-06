# scheduler

kubernets 在节点上由kubelet对容器进行生命周期管理，每当kubelet监测到pod/container的状态发生变化是，会进行容器平台接口的调用，在分发pod上，kubelet调用的是kube-scheduler，调用方式为apiServer。

kube-scheduler也是一个pod，它的默认配置文件是

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    image: k8s.gcr.io/kube-scheduler:v1.18.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}
```

可以看到，Scheduler调度器运行在master节点，它的核心功能是监听apiserver来获取PodSpec.NodeName为空的pod，然后为pod创建一个binding指示pod应该调度到哪个节点上，调度结果写入apiserver。

# scheduler extender

scheduler extender是scheduler的外部扩展，用户可以根据需求自行建立调度服务，并用apiserver进行访问。我们不需要为extender创建yaml来启动它，只需要修改原有的yaml设置以满足extender的启动条件即可！

配置文件中增加了关于extender和extender policy的设置

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --config=/etc/kubernetes/scheduler-extender.yaml
    - --v=9
    image: gcr.azk8s.cn/google_containers/kube-scheduler:v1.16.2
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /etc/kubernetes/scheduler-extender.yaml
      name: extender
      readOnly: true
    - mountPath: /etc/kubernetes/scheduler-extender-policy.json
      name: extender-policy
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
    path: /etc/kubernetes/scheduler.conf
    type: FileOrCreate
  name: kubeconfig
  - hostPath:
    path: /etc/kubernetes/scheduler-extender.yaml
    type: FileOrCreate
  name: extender
  - hostPath:
    path: /etc/kubernetes/scheduler-extender-policy.json
    type: FileOrCreate
  name: extender-policy
status: {}
```



本项目中，我们首先要建立在scheduler和extender间建立http通信传输json，这里预留了127.0.0.1:8080/的地址，并准备了priority和filter两个api。

extender将scheduler由http request传入的json解码，读取节点信息，产生节点优先级和过滤节点信息后，得到extenderFilterResult和hostPriorityList，最后extender会将它们打包成json，由http response返回给scheduler，进行调度。



