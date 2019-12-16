## 系统环境

- **kubernetes 版本：1.14.0**
- **jenkins 版本：2.172**
- **jenkins 部署示例文件 Github 地址：https://github.com/my-dlq/blog-example/tree/master/jenkins-deploy**



## 一、设置存储目录

**在 Kubenetes 环境下所起的应用都是一个个 Docker 镜像，为了保证应用重启的情况下数据安全，所以需要将 Jenkins 持久化到存储中。这里用的是 NFS 网络存储，方便在 Kubernetes 环境下应用启动节点转义数据一致。当然也可以选择存储到本地，但是为了保证应用数据一致，需要设置 Jenkins 固定到某一 Kubernetes 节点**



#### 1、安装 NFS 服务器

```shell
$ yum install -y nfs-common nfs-utils rpcbind mkdir /nfsdata
$ chmod 666 /nfsdata
$ chown nfsnobody /nfsdata
$ cat /etc/exports
    /nfsdata *(rw,no_root_squash,no_all_squash,sync)
$ systemctl start rpcbind
$ systemctl start nfs
```



#### 2、挂在 NFS 并设置存储文件

**如果不能直接操作 NFS 服务端创建文件夹，需要知道 NFS 服务器地址，然后将其挂在到本地目录，进入其中创建 Jenkins 目录空间**

**(1)、挂载 NFS**

```shell
$ mount -o vers=4.1 192.168.2.11:/nfs/ /nfs
```

**(2)、在 NFS 共享存储文件夹下创建存储 Jenkins 数据的文件夹**

```shell
$ mkdir -p /nfs/data/jenkins
```



## 二、创建 PV & PVC

**创建 PV 绑定 NFS 创建的 Jenkins 目录，然后创建 PVC 绑定这个 PV，将此 PVC 用于后面创建 Jenkins 服务时挂载的存储**



#### 1、准备 PV & PVC 部署文件

> 一定要确保 PV 的空间大于 PVC，否则无法关联，创建  jenkins-pv-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins
  labels:
    app: jenkins
spec:
  capacity:          
    storage: 10Gi
  accessModes:       
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  
  mountOptions:   #NFS挂在选项
    - hard
    - nfsvers=4.1    
  nfs:            #NFS设置
    path: /nfs/data/jenkins   
    server: 192.168.2.11
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi			#生产环境空间一定要设置比较大点
  selector:
    matchLabels:
      app: jenkins
```



#### 2、创建 PV & PVC

> 提前将 namespace 修改成你自己的 namespace

- **-n：指定 namespace**

```shell
$ kubectl apply -f jenkins-pv-pvc.yaml -n public
```



## 三、创建 ServiceAccount & ClusterRoleBinding

**此 kubernetes 集群用的是 RBAC 安全插件，必须创建权限给一个 ServiceAccount，然后将此 ServiceAccount 绑定到 Jenkins 服务，这样赋予 Jenkins 服务一定权限执行一些操作，为了方便，这里将 cluster-admin 绑定到 ServiceAccount 以保证 Jenkins 能拥有一定的权限**

> 注意：请提前修改 yaml 中的 namespace

#### **1、jenkins-rbac.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin       #ServiceAccount名
  namespace: mydlqcloud     #指定namespace，一定要修改成你自己的namespace
  labels:
    name: jenkins
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins-admin
  labels:
    name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: mydlqcloud
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

**(2)、创建 RBAC 命令**

```
$ kubectl create -f jenkins-rbac.yaml
```



## 四、创建 Service & Deployment

**这里开始部署 Jenkins 服务，创建 Service 与 Deployment，其中 Service 暴露两个接口 80880 与 50000。而 Deployment 里面要注意的是要设置上面创建的 ServiceAccount ，并且设置容器安全策略为“runAsUser: 0”以 Root 权限运行容器，而且暴露8080、50000两个端口**



#### 1、创建 Service & Deployment 部署文件

> jenkins-deployment.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  labels:
    app: jenkins
spec:
  type: NodePort
  ports:
  - name: http
    port: 8080          #服务端口
    targetPort: 8080
    nodePort: 32001		#NodePort方式暴露 Jenkins 端口
  - name: jnlp
    port: 50000         #代理端口
    targetPort: 50000
    nodePort: 32002
  selector:
    app: jenkins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  labels:
    app: jenkins
spec:
  selector:
    matchLabels:
      app: jenkins
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-admin
      containers:
      - name: jenkins
        image: registry.cn-shanghai.aliyuncs.com/mydlq/jenkins:2.172
        securityContext:                     
          runAsUser: 0       #设置以ROOT用户运行容器
          privileged: true   #拥有特权
        ports:
        - name: http
          containerPort: 8080
        - name: jnlp
          containerPort: 50000
        resources:
          limits:
            memory: 2Gi
            cpu: "1000m"
          requests:
            memory: 1Gi
            cpu: "500m"
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: "JAVA_OPTS"  #设置变量，指定时区和 jenkins slave 执行者设置
          value: "
                   -Xmx$(LIMITS_MEMORY)m
                   -XshowSettings:vm
                   -Dhudson.slaves.NodeProvisioner.initialDelay=0
                   -Dhudson.slaves.NodeProvisioner.MARGIN=50
                   -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
                   -Duser.timezone=Asia/Shanghai
                 "    
        - name: "JENKINS_OPTS"
          value: "--prefix=/jenkins"         #设置路径前缀加上 Jenkins
        volumeMounts:                        #设置要挂在的目录
        - name: data
          mountPath: /var/jenkins_home
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: jenkins      #设置PVC
```



**参数说明**：

- **JAVA_OPTS：** JVM 参数设置

- **JENKINS_OPTS：** Jenkins 参数设置

- **设置执行任务时候不等待：**

  默认情况下，Jenkins生成代理是保守的。例如，如果队列中有两个构建，它不会立即生成两个执行器。它将生成一个执行器，并等待某个时间释放第一个执行器，然后再决定生成第二个执行器。Jenkins确保它生成的每个执行器都得到了最大限度的利用。如果你想覆盖这个行为，并生成一个执行器为每个构建队列立即不等待，所以在Jenkins启动时候添加这些参数:

  - -Dhudson.slaves.NodeProvisioner.initialDelay=0
  - -Dhudson.slaves.NodeProvisioner.MARGIN=50
  - -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85



#### 2、部署 Jenkins

> 执行 Kuberctl 命令将 Jenkins 部署到 Kubernetes 集群，注意：将“-n”后面的 namespace 换成你自己的 namespace

- **-n：指定应用启动的 namespace**

```shell
$ kubectl create -f jenkins-deployment.yaml -n mydlqcloud
```



## 五、获取 Jenkins 生成的 Token

> 在安装 Jenkins 时候，它默认生成一段随机字符串，用于安装验证。这里访问它的安装日志，获取它生成的 Token 字符串



#### (1)、查看 Jenkins Pod 启动日志**

> 注意：这里“-n”指的是要 namespace，后面跟的 namespace 请替换成你jenkins 启动的 namespace

```shell
$ kubectl log $(kubectl get pods -n mydlqcloud | awk '{print $1}' | grep jenkins) -n mydlqcloud
```



#### (2)、查看日志中生成的 Token 字符串

> 查看日志，默认给的token为：

```shell
*************************************************************
*************************************************************
*************************************************************
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

3b3e0dda9d6746358ade987775f924ef

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
*************************************************************
*************************************************************
*************************************************************
```



## 六、启动 Jenkins 进行安装

> 输入集群地址和 Jenkins Service 提供的 NodePort 端口，访问 Jenkins 进行安装步骤，可以按下一步步执行：



#### 1、进入Jenkins

**输入集群地址和上面设置的Nodeport方式的端口 32001，然后输入上面获取的 Token 字符串。例如，本人的集群IP为 192.168.2.11 ，所以就可以访问 http://192.168.2.11:32001/jenkins ，进入后可以看到下面的界面**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6h8RjLicUrLNKf2nJW5fFdcFhiadic4ibVTyZK4p6OlHBYE7Z0fdtGFXIYQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



#### 2、安装插件

> 选择自定义插件来进行安装

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6ef4M0Obsob5ib0XU2nJCaYJlQ5qkZRPXM0LTIEkuayd9k2LoHvK0mag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**安装一些常用的插件，这里可以选择一下，推荐安装下面插件**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6qb6In0msicYC6iccppPKhw7EEH659gR34ibJJENpp5SRUeyaVEqxAGoeA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6B6icyNiasbxeUGDb9aWoibzG31WVFfK7UaKfRcic6ojiau4OCJ3YsUola1Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6KyNicLhzAicp3ahQia79ia8ZnPoxFfsp0QibViaCE9B3icYAlDftB5UBLiceHQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6YvVPnibRv7AH1SvkPY93Ob5MtEgozt4ml1OKnUCd6ITdo7aOJPb76wA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**确定后可以看到正在安装插件界面**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6nzQO3VkiaWzuiaMSUdibSzHibBVp2GrfI7GRr83ibecwy47JytsbvoN7icrg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



#### 3、设置用户名、密码

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6su21y2UrwJezucDjYk1vBjb6diaKgJSMQxM5MRKpW1Az00UAicrOd0xA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**然后进入 Jenkins 界面**

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6YXNMTtEA0K4Dm22cLicD6H34KOWWDcksxLqIgkhFbefGicRezW7IuHCg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 七、其他

#### 1、插件因网络问题安装失败

> 如果 Jenkins 安装插件失败，则可以按一下设置

- **(1)、进入地址 http://192.168.2.11:32001/jenkins/pluginManager/advanced**

- **(2）、将最下面的 Update Site 的 URL 地址替换成：http://mirror.esuni.jp/jenkins/updates/update-center.json**

- **(3)、点“submit”按钮，然后点右下角角 “check now”**

- **(4)、然后输入地址 http://192.168.2.11:32001/jenkins/restart 重启 jenkins 后再重新安装插件**

  

> PS：上面地址替换成你们的集群地址及端口。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6PibLFAYBRF0lgJjQ5VthOWefiaBSCS3a3VySvATsR1XXAS66WtD8P0wQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_jpg/7lxf8j18x2LGOsCNuQya47UoXKGLibgC6Vd7zBEAaz4lkcZzP1oQNcNeQJU22ibS2hWU9xUK8eXxibbqYugQmJmQg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)