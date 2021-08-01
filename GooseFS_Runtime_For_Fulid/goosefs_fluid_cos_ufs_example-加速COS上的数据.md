## 前提条件


- [Fluid](https://github.com/fluid-cloudnative/fluid)(version >= 0.6.0)
  
- 请参考[安装文档](./jindo_fluid_install.md)完成安装
## 创建 dataset 和 GooseFSRuntime


创建一个 resource.yaml 文件里面包含两部分：

- 首先包含数据集及 ufs 的 dataset 信息，创建一个 Dataset CRD 对象，其中描述了数据集的来源，如例子中的 test-bucket。
- 接下来需要创建一个 GooseFSRuntime，相当于启动一个 GooseFS 的集群来提供缓存服务。



```yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hadoop
spec:
  mounts:
    - mountPoint: cos://test-bucket/
      options:
        fs.cos.accessKeyId: <COS_ACCESS_KEY_ID>
        fs.cos.accessKeySecret: <COS_ACCESS_KEY_SECRET>
        fs.cos.endpoint: <COS_ENDPOINT> 
      name: hadoop
---
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hadoop
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: HDD
        path: /mnt/disk1
        quota: 100G
        high: "0.9"
        low: "0.2"
```


当然为了AK等信息安全性，建议使用secret来保存相关密钥信息，secret使用请参考[使用参数加密](./jindo_fluid_encryptOption.md)


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
stringData:
  fs.cos.accessKeyId: <COS_ACCESS_KEY_ID>
  fs.cos.accessKeySecret: <COS_ACCESS_KEY_SECRET>
---
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hadoop
spec:
  mounts:
    - mountPoint: cos://yourbucket/
      options:
        fs.cos.endpoint: cos.<region>.myqcloud.com
      name: hadoop
      encryptOptions:
        - name: fs.cos.accessKeyId
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: fs.cos.accessKeyId
        - name: fs.cos.accessKeySecret
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: fs.cos.accessKeySecret
---
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hadoop
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: SSD
        path: /mnt/disk1
        quota: 100G
        high: "0.9"
        low: "0.2"
```


- mountPoint：表示挂载UFS的路径，路径中不需要包含 endpoint 信息。
- fs.cos.accessKeyId/fs.cos.accessKeySecret：cos bucket的AK信息，有权限访问该 bucket。
- fs.cos.endpoint：cos bucket的endpoint信息，公网或内网地址皆可，内网地址使用条件为您的k8s集群所在区域和cos区域相同。
- replicas：表示创建 GooseFS 集群节点的数量。
- mediumtype： GooseFS 暂只支持HDD/SSD/MEM中的其中一种。
- path：存储路径，暂只支持一块盘，当选择MEM做缓存也需要一块盘来存储log等文件。
- quota：缓存最大容量，单位G。
- high：水位上限大小 / low： 水位下限大小。



创建 GooseFSRuntime


```shell
kubectl create -f resource.yaml
```


查看部署的 GooseFSRuntime 情况，显示都为 Ready 状态表示部署成功。
```shell
kubectl get goosefsruntime hadoop
NAME     MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
hadoop    Ready           Ready           Ready     62m
```


查看 dataset 的情况，现实 Bound 状态表示 dataset 绑定成功。


```shell
$ kubectl get dataset hadoop
NAME     UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hadoop        511MiB       0.00B    180.00GiB              0.0%          Bound   1h
```


查看 PV，PVC 创建情况，GooseFSRuntime部署过程中会自动创建PV和PVC
```shell
kubectl get pv,pvc
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/hadoop   100Gi      RWX            Retain           Bound    default/hadoop                           58m

NAME                           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/hadoop   Bound    hadoop   100Gi      RWX                           58m
```
### 检查服务是否正常！！！
此步骤建议用户创建成功后一定要进行验证

#### 检查底层存储端点是否成功连接
1、登陆到 master/worker pod上
```shell
kubectl get pod
NAME                              READY   STATUS      RESTARTS   AGE
hadoop-goosefs-fuse-svz4s         1/1     Running     0          23h
hadoop-goosefs-master-0           1/1     Running     0          23h
hadoop-goosefs-worker-2fpbk       1/1     Running     0          23h
```
```shell
kubectl exec -ti hadoop-goosefs-master-0 bash
hadoop fs -ls jfs://jindo/
```
观察是否可以正常list文件

2、登陆到 fuse pod上
```shell
kubectl exec -ti hadoop-goosefs-fuse-svz4s bash
ls /jfs/jindo/
```
观察是否可以正常list文件