在 Fluid 中可以通过 nodeselector 来指定 master 节点的部署，比如选择部署 master 到性能较好的 k8s 机器上。


本文档将向你简单地展示上述特性
## 前提条件


在运行该示例之前，请参考[安装文档]()完成安装完成安装，并检查Fluid各组件正常运行：


```shell
$ kubectl get pod -n fluid-system
goosefsruntime-controller-5b64fdbbb-84pc6   1/1     Running   0          8h
csi-nodeplugin-fluid-fwgjh                  2/2     Running   0          8h
csi-nodeplugin-fluid-ll8bq                  2/2     Running   0          8h
dataset-controller-5b7848dbbb-n44dj         1/1     Running   0          8h
GooseFSruntime-controller-654fb74447-cldsv    1/1     Running   0          8h
```


通常来说，你会看到一个名为`dataset-controller`的Pod、一个名为 `goosefsruntime-controller` 的Pod和多个名为`csi-nodeplugin`的Pod正在运行。其中，`csi-nodeplugin`这些Pod的数量取决于你的 Kubernetes 集群中结点的数量。
## 新建工作环境


```shell
$ mkdir <any-path>/co-locality
$ cd <any-path>/co-locality
```
## 示例

**查看全部结点**


```shell
$ kubectl get nodes
NAME                       STATUS   ROLES    AGE     VERSION
ap-beijing.192.168.1.146   Ready    <none>   7d14h   v1.16.9-aliyun.1
ap-beijing.192.168.1.147   Ready    <none>   7d14h   v1.16.9-aliyun.1
```


**使用标签标识结点**


```shell
$ kubectl label nodes ap-beijing.192.168.1.146 hbase-cache=true
```


**再次查看结点**


```shell
$ kubectl get node -L hbase-cache
NAME                       STATUS   ROLES    AGE     VERSION            HBASE-CACHE
ap-beijing.192.168.1.146   Ready    <none>   7d14h   v1.16.9-aliyun.1   true
ap-beijing.192.168.1.147   Ready    <none>   7d14h   v1.16.9-aliyun.1
```


**检查待创建的Dataset资源对象**


```yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: cos://test-bucket/
      options:
        fs.cos.accessKeyId: <COS_ACCESS_KEY_ID>
        fs.cos.accessKeySecret: <COS_ACCESS_KEY_SECRET>
        fs.cos.endpoint: <COS_ENDPOINT> 
      name: hbase
```

**创建Dataset资源对象**


```shell
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
```


**检查待创建的GooseFSRuntime资源对象**


```yaml
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hbase
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2G
        high: "0.8"
        low: "0.7"
  master:
    nodeSelector:
      hbase-cache: true
```


该配置文件片段中，包含了许多与GooseFS相关的配置信息，这些信息将被Fluid用来启动一个GooseFS实例。上述配置片段中的`spec.replicas`属性被设置为1,这表明Fluid将会启动一个包含1个GooseFS Master和1个GooseFS Worker的GooseFS实例

**创建GooseFSRuntime资源并查看状态**


```shell
$ kubectl create -f runtime.yaml
alluxioruntime.data.fluid.io/hbase created

$ kubectl get pod -o wide
NAME                 READY   STATUS    RESTARTS   AGE    IP              NODE                       NOMINATED NODE   READINESS GATES
hbase-goosefs-fuse-42csf     1/1     Running   0          104s   192.168.1.146   cn-beijing.192.168.1.146   <none>           <none>
hbase-goosefs-master-0       2/2     Running   0          3m3s   192.168.1.147   cn-beijing.192.168.1.146   <none>           <none>
hbase-goosefs-worker-l62m4   2/2     Running   0          104s   192.168.1.146   cn-beijing.192.168.1.146   <none>           <none>
```


在此处可以看到，master成功启动并且运行在具有指定标签（即`hbase-cache=true`）的结点之上。
