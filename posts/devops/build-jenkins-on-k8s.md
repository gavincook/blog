# 在k8s中搭建Jenkins

## 1. 背景
原本Jenkins使用Docker在某台虚拟机进行搭建，此种方式有几个缺点：

* 需要单独的虚拟机资源，虚拟机的维护成本更高
* Jenkins服务的IP出口也只能使用虚拟机IP
* 不易水平伸缩

## 2. 搭建
### 2.1 Jenkins搭建
1. 首先添加Jenkins Helm源

    ```bash
    helm repo add jenkins https://charts.jenkins.io
    helm repo update
    ```
2. 自定义配置values.yaml
默认的values.yaml参见[链接](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/values.yaml)，其中的配置项说明参见：[VALUES_SUMMARY](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/VALUES_SUMMARY.md)
其中主要需要自定义配置的如下：

    ```bash
    controller.servicePort: 80 //暴露的服务端口，默认为8080
    serviceType: LoadBalancer //使用公网IP入口暴露Jenkins，也可使用Ingress进行管控
    persistence.storageClass: "managed-csi-premium" //不设置则为集群设置的默认storageClass
    persistence.size: "128Gi"//磁盘大小
    ```
aks中默认有的storageclass有两种：
* file.csi.azure.com：文件，也即使用的是Azure Blob作为底层的存储引擎，支持`ReadWriteMany`挂载。写操作较慢，适用少写多读的场景。
* disk.csi.azure.com: 磁盘，使用的是Azure Disk作为底层存储引擎，默认不支持`ReadWriteMany`挂载。若需要，则需要单独申明PVC。注意磁盘的多读写挂载，受限于磁盘最大共享数。具体参见：[disk share](https://docs.microsoft.com/en-us/azure/virtual-machines/disks-shared-enable?tabs=azure-portal#premium-ssd-ranges)

3. 使用Helm搭建
我们这里将jenkins安装到命名空间`devops`
    ```bash
    chart=jenkinsci/jenkins
    helm install jenkins -n devops -f jenkins-values.yaml --version 3.11.4 $chart
    ```
部署后，可查看`devops`命名空间下的`LoadBalancer`的IP，访问`http://{IP}`即可。

4. Jenkins初始化密码
    ```bash
    jsonpath="{.data.jenkins-admin-password}"
    secret=$(kubectl get secret -n devops jenkins -o jsonpath=$jsonpath)
    echo $(echo $secret | base64 --decode)
    ```
然后使用账号`admin`和上述获取到的密码进行登录即可。

### 2.2 k8s cloud node
helm安装后会自动增加一个k8s的cloud node配置，其中需要修改的配置主要有：

* Docker Image
  修改jenkins agent镜像，默认使用`jenkins/inbound-agent:xxx`，但平常我们构建时通常需要一些更多的软件依赖环境，比如：`docker-in-docker`用于动态构建需要依赖的容器。可以基于`https://github.com/jenkinsci/docker-inbound-agent`进行自定义镜像改造。
* Command to run
  设置为空即可
* Limit CPU / Limit Memory
  调整agent的资源，若有必要
* ImagePullSecrets
  若需要从私有仓库拉取镜像，需要配置私有仓库的秘钥，填写秘钥的名称即可。创建方式为：

  ```
kubectl create secret docker-registry {秘钥名称} --docker-server={server.url} --docker-username={username} --docker-password={password} --docker-email={user.email} -n {namespace}
  ```

* Service Account
  配置为Values.yaml中的SA配置，默认为`jenkins`

* Pod Template
  其中的Lables的名称用于Jenkins通过标签进行选择时使用。若界面配置无法满足对Pod Template自定义，则可使用`Raw YAML for the Pod`部分进行Pod Template编写（格式为yaml）

## 3. 善后
当k8s cloud node配置完后，现在就可以直接开始使用Jenkins了。为了保障以后的顺畅使用，还有几点需要善后：

1. 删除secret: `jenkins`，避免后续Jenkins重启之后，管理员密码被重置
2. 将configMap：`jenkins-jenkins-jcasc-config`中的k8s cloud node配置进行删除（`clouds`部分），避免后续Jenkins重启后，k8s cloud node配置会重置为安装后的默认配置。
