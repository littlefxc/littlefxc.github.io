---
title: 基于Jenkins、SVN、Harbor、Helm和Kubernetes的持续集成
date: 2020-04-28 13:59:44
tags: [k8s,jenkins,helm,svn,CI/CD,devops]
categories: devops
---

本文介绍如何结合 Kubernetes + Jenkis + SVN + Harbor + Helm 实现一个完整的 CI/CD 流水线作业进行持续集成和持续部署。

<!-- more -->

# 1 安装 Helm

下载 Helm3，同时添加 chart 库

[下载地址](https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz)

```bash
$ tar -zxvf helm-v3.1.2-linux-amd64.tar.gz
$ ln -s /home/k8s-projects/helm-projects/linux-amd64/helm /usr/local/bin/helm
$ helm repo add stable http://mirror.azure.cn/kubernetes/charts
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

# 2 使用 Kubernetes 的动态存储

尽管通过 hostPath 或者 emptyDir 的方式可以用来来持久化我们的数据，但是显然我们还需要更加可靠的存储来保存应用的持久化数据，这样容器在重建后，依然可以使用之前的数据。但是显然存储资源和 CPU 资源以及内存资源有很大不同，为了屏蔽底层的技术实现细节，让用户更加方便的使用，Kubernetes 便引入了 PV 和 PVC 两个重要的资源对象来实现对存储的管理。

本文中我们使用 StorageClass，要使用它就得安装对应的自动配置程序，比如我们这里存储后端使用的是 nfs，那么我们就需要使用到一个 nfs-client 的自动配置程序，我们也叫它 Provisioner，这个程序使用我们已经配置好的 nfs 服务器，来自动创建持久卷，也就是自动帮我们创建 PV。

## 2.1 **CentOS 7 下 yum 安装和配置 NFS**

{% post_link CentOS7下yum安装和配置NFS CentOS7下yum安装和配置NFS %}

## 2.2 安装 nfs-client-provisioner

```bash
$ helm install nfs-client-provisioner-dev stable/nfs-client-provisioner --set nfs.server=192.168.200.19 \
--set nfs.path=/home/k8s-projects/k8s-nfs --set storageClass.defaultClass=true
```

查看安装结果：

```bash
$ kubectl get storageClass -n dev
NAME                   PROVISIONER                                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-client-provisioner-dev   Delete          Immediate           true                   13d
```

可以看到 `storageClass` 顺利创建。

# 3 k8s 中安装 Jenkins

使用 Helm 安装 jenkins，命名空间为 `dev`，指定存储 `storageClass` 为 `nfs-client` 

```bash
$ helm pull stable/jenkins
$ helm install jenkins-dev /home/k8s-projects/helm-projects/jenkins --namespace dev \
--set master.serviceType=NodePort --set persistence.storageClass=nfs-client --set master.adminPassword=awifi@123
```

我们利用 Kubernetes 来动态运行 Jenkins 的 Slave 节点，可以很好的来解决传统的 Jenkins Slave 浪费大量资源的缺点。

# 4 k8s 中安装 Redis 集群

使用 Helm 安装 Redis 集群， 命名空间为 `dev`, 指定存储`storageClass`为`nfs-client`, 密码设置为`123456`

```bash
$ helm pull redis-cluster-dev bitnami/redis-cluster
$ helm install redis-cluster-dev /home/k8s-projects/helm-projects/redis-cluster --namespace=dev \
--set global.storageClass=nfs-client,global.redis.password=123456
```

# 5 k8s 中配置外部的 mysql

一般来说，k8s 集群外部的服务，集群内是无法访问的。

以 mysql 为例，我们想要部署的服务中想使用现有的 mysql 数据库，k8s也有解决办法，将外部服务作为 `Endpoints`，同时以 `Service` 的形式暴露到集群内。

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service-mysql
subsets:
  - addresses:
    - ip: 192.168.212.59
    ports:
    - port: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: external-service-mysql
spec:
  ports:
  - port: 3306
```

# 6 持续构建流程

![在这里插入图片描述](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70.png)


1. 开发人员提交代码到 SVN 代码仓库
2. Jenkins 触发构建构建任务，根据 Pipeline 脚本定义分步骤构建
3. 先进行代码检出，单元测试
4. 然后进行 Maven 构建（Java 项目）
5. 根据构建结果构建 Docker 镜像
6. 推送 Docker 镜像到 Harbor 仓库
7. 触发更新服务阶段，使用 Helm 安装/更新 Release
8. 查看服务是否更新成功。

# 7 Docker 镜像构建

以用户数据服务为例。

由于我们要将项目部署到 `Kubernetes` 集群中去，所以我们需要将服务端进行容器化，所以我们在项目根目录下面添加一个 `Dockerfile` 文件进行镜像构建：

```docker
FROM openjdk:8-jdk-alpine

MAINTAINER fengxuechao <fengxuechao.littlefxc@gmail.com>

ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8 TZ=Asia/Shanghai
RUN mkdir /app /app/config /app/logs

WORKDIR /app

COPY ./*.jar /app/app.jar

EXPOSE 8080
```

由于服务端代码是基于Spring Boot构建的，所以我们这里使用一个`openjdk`的基础镜像，将打包过后的jar包放入镜像之中，然后通过启动容器使用`java -jar`命令直接启动即可，这里就会存在一个问题了，我们是在 Jenkins 的 Pipeline 中去进行镜像构建的，这个时候项目中并没有打包好的jar包文件，那么我们应该如何获取打包好的jar包文件呢？这里我们可以使用两种方法：

第一种就是如果你用于镜像打包的 Docker 版本大于`17.06`版本的话，那么你可以使用 Docker 的多阶段构建功能来完成镜像的打包过程，我们只需要将上面的Dockerfile文件稍微更改下即可，将使用maven进行构建的工作放到同一个文件中：

```docker
FROM maven:3.6-alpine as BUILD

# 拷贝源码
COPY src /usr/app/src
COPY pom.xml /usr/app
# 拷贝自定义 maven settings.xml
COPY settings.xml /usr/share/maven/ref/

RUN mvn -f /usr/app/pom.xml clean package -Dmaven.test.skip=true

FROM openjdk:8-jdk-alpine

MAINTAINER fengxuechao <fengxuechao.littlefxc@gmail.com>

ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8 TZ=Asia/Shanghai

RUN mkdir /app

WORKDIR /app

COPY --from=BUILD /usr/app/target/*.jar /app/app.jar

EXPOSE 8080
```

这里我们定义了两个阶段，第一个阶段利用`maven:3.6-alpine`这个基础镜像将我们的项目进行打包，然后将该阶段打包生成的jar包文件复制到第二阶段进行最后的镜像打包，这样就可以很好的完成我们的 Docker 镜像的构建工作。

第二种方式就是我们传统的方式，在 Jenkins Pipeline 中添加一个maven构建的阶段，然后在第二个 Docker 构建的阶段就可以直接获取到前面的jar包了，也可以很方便的完成镜像的构建工作，为了更加清楚的说明 Jenkins Pipeline 的用法，我们这里采用这种方式，所以 Dockerfile 文件还是使用第一个就行。

# 8 Jenkins

## 8.1 配置

首先，我们需要去 Jenkins -> Manage Jenkins -> Manage Plugins, 检查插件 `Kubernetes`, `Subversion`有没有安装，没有就安装一下，这两个插件是后面一系列操作的基础。

现在项目准备好了，接下来我们可以开始 Jenkins 的配置，我们想要完全自由灵活的配置我们的Jenkins Slave 节点，我们直接直接在 Pipeline 中去自定义 Slave Pod 中所需要用到的容器模板，这样我们需要什么镜像只需要在 Slave Pod Template 中声明即可，完全不需要去定义一个庞大的 Slave 镜像了。

首先去掉 Jenkins 中 kubernetes 插件中的 Pod Template 的定义，Jenkins -> Manage Jenkins -> Manage Nodes and Clouds -> Configure Clouds -> Kubernetes区域，删除下方的`Kubernetes Pod Template` -> 保存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509194819239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

点击按钮"连接测试"。如果出现错误，八成是权限问题，建议你去查看一下官网资料[使用 RBAC 鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)。

## 8.2 创建一个流水线任务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509194819236.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

## 8.3 创建3个变量

然后选中 "This project is parameterized", 添加 3 个 "Boolean Parameter". 分别是 `helmDelete`, `helmInstall` 和 `helmUpgrade`, 其中默认选中 `helmUpgrade`.

这 3 个变量与我们自定义的 Jenkinsfile 脚本有关，是必须要创建的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509194819215.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

## 8.4 配置 Pipeline

我们的 Jenkins 的任务是有 Jenkinsfile 推动的.

然后在下面的流水线区域我们可以选择`Pipeline script`然后在下面测试流水线脚本，我们这里选择`Pipeline script from SCM`，意思就是从代码仓库中通过`Jenkinsfile`文件获取Pipeline script脚本定义，然后选择 SCM 来源为 Subversion(如果没有，先去安装这个插件)，在出现的列表中配置上仓库地址 http://.../docker，由于我们是在一个 Slave Pod 中去进行构建，所以如果使用 SSH 的方式去访问 Gitlab 代码仓库的话就需要频繁的去更新 SSH-KEY，所以我们这里采用直接使用用户名和密码的形式来方式：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509200524318.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70#pic_center)

在`Credentials`区域点击添加按钮添加我们访问 SVN 的用户名和密码：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509194818424.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

配置成功后我们只需要往 SVN 仓库推送代码,然后手动点击触发 Pipeline 构建了。接下来我们直接在服务端代码仓库目录下面添加Jenkinsfile文件，用于描述流水线构建流程。

## 8.5 Jenkinsfile 示例

首先要说明的是 Jenkinsfile 是基于 Groovy 语言的，与Java有很多的相似性。

`def` 是在Groovy用来定义标识符的关键字.在使用它的时候请注意 `def` 声明的变量所对应的是否符合你要编译和部署的对象。

- `label`标签的定义，我们这里使用 UUID 生成一个随机的字符串，这样可以让 Slave Pod 每次的名称都不一样，而且这样就不会被固定在一个 Pod 上面了，以后有多个构建任务的时候就不会存在等待的情况了。
- `dockerRegistryUrl` 中是公司docker镜像私服地址。
- `mavenProjects` 是项目的根目录
- `svnRemote` 是项目的SVN地址
- `appName` 是maven编译后的jar包名，和`imageTag` 一起定位 jar 在maven本地仓库中的地址、docker镜像名组成部分，同时也是helm中的项目名
- `imageTag` 是maven编译后的jar包版本，和`appName` 一起定位 jar 在maven本地仓库中的地址、docker镜像名组成部分
- `namespace` 是docker镜像在公司私服harbor中的项目名，可以通过 [http://alpha-harbor.51iwifi.com/harbor/projects](http://alpha-harbor.51iwifi.com/harbor/projects) 查看项目名是否存在。
- `image` 是docker镜像编译打包后的镜像名，由 `appName` 和`imageTag` 组成
- `mvnBuildModules` 是maven编译的模块，格式为 groupId:artifactId。请确保你要编译的maven工程包含在该模块下。
- `chartDir` 通用 spring-cloud 应用 k8s helm 部署模板, 你也可以自定义！
- `svnHelm` 通用 spring-cloud 应用 k8s helm 部署模板的远程仓库地址
- `dockerPath` 是源码检出后在Jenkins Slave Pod 中拥有Jenkinsfile的相对地址，docker镜像编译的工作目录。
- `jenkinsSVNCredentialsId` 是你在本文上一章节中定义的Jenkins中的svn凭据，请确保这个凭据定义的用户拥有访问 `svnRemote` 和 `svnHelm` 这两个变量定义的远程仓库的权限。

我们使用`podTemplate`来定义不同阶段使用的的容器，阶段总共分为5个阶段：

代码检出 → 单元测试 → Maven 打包 → Docker 镜像构建/推送 → Helm 安装/更新服务。

代码检出在默认的 Slave 容器中即可；单元测试我们这里输出一些期望目标后直接忽略，有需要这个阶段的同学自己添加上自己的测试即可；Maven 打包肯定就需要 maven 的容器了；Docker 镜像构建/推送需要 Docker 环境；最后的 Helm 更新服务是不是就需要一个有 Helm 的容器环境了，所以我们这里就可以很简单的定义podTemplate了，如下定义：

```groovy
def label = "jenkins-slave-${UUID.randomUUID().toString()}"

def dockerRegistryUrl = "192.168.195.2";

// 项目的根目录
def mavenProjects = "open-platform-dataservice";
def svnRemote = "http://svn.51iwifi.com/repos/AWIFI-PROJECT/capacitygroup/code/trunk/${mavenProjects}";

// 请填写maven编译后的jar包名
def appName = "opf-dataservice-user-provider";
// 请填写maven编译后的jar包版本
def imageTag = "1.0.10";

def namespace = "dev";
def image = "${namespace}/${appName}";

// 请填写要maven编译的模块，格式为 groupId:artifactId
def mvnBuildModules = "com.awifi.capacity:open-platform-dataservice-user";

// 请选择在配置在Jenkins 中的 SVN 凭据
def jenkinsSVNCredentialsId = "SVN-fengxc";
// 通用 spring-cloud 应用 k8s helm 部署模板, 你也可以自定义！
def chartDir = "helm-spring-cloud"
def svnHelm = "http://svn.51iwifi.com/repos/AWIFI-PROJECT/capacitygroup/code/trunk/${chartDir}";

// 请填写包含 Jenkinsfile,Dockerfile 的相对路径
def dockerPath = "${mavenProjects}/open-platform-dataservice-user/open-platform-dataservice-user-provider/docker";

podTemplate(label: label, containers: [
        containerTemplate(name: 'maven', image: 'maven:3.6-alpine', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'helm-kubectl', image: 'dtzar/helm-kubectl:3.1.2', command: 'cat', ttyEnabled: true)
], volumes: [
        nfsVolume(mountPath: '/root/.m2', readOnly: false, serverAddress: '192.168.200.19', serverPath: '/home/k8s-projects/k8s-nfs/m2'),
        hostPathVolume(mountPath: '/home/jenkins/.kube', hostPath: '/root/.kube'),
        hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
    node(label) {
       stage('代码检出') {
            echo "1. 代码检出阶段";
            sh """
               rm -fr ./*
               mkdir ${mavenProjects}
               mkdir ${chartDir}
               """
            echo "检出项目源码"
            checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: "${jenkinsSVNCredentialsId}", depthOption: 'infinity', ignoreExternalsOption: true, local: "${mavenProjects}", remote: "${svnRemote}"]], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            echo "检出 helm chart"
            checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: "${jenkinsSVNCredentialsId}", depthOption: 'infinity', ignoreExternalsOption: true, local: "${chartDir}", remote: "${svnHelm}"]], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
        }
        stage('单元测试') {
            echo "2. 测试阶段";
            echo "项目 SVN 地址: ${svnRemote}"
            echo "maven 编译模块目标: ${mvnBuildModules}"
            echo "Docker 构建目标: ${dockerRegistryUrl}/${image}"
            echo "Helm 模板地址: ${svnHelm}"
            echo "Helm 部署目标: namespace = ${namespace}, name = ${appName}"
            echo "Helm 删除应用：${helmDelete}"
            echo "Helm 部署应用：${helmInstall}"
            echo "Helm 升级应用：${helmUpgrade}"
        }
        stage('代码编译打包') {
            try {
                container('maven') {
                    echo "3. 代码编译打包阶段"
                    sh """
                       cd ${mavenProjects}
                       mvn clean install -pl ${mvnBuildModules} -am -Dmaven.test.skip=true
                       """
                }
            } catch (exc) {
                println "构建失败 - ${currentBuild.fullDisplayName}"
                throw (exc)
            }
        }
        stage('构建 Docker 镜像') {
            withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                container('docker') {
                    echo "4. 构建 Docker 镜像阶段"
                    sh """
                       cd ${dockerPath}
                       cp /root/.m2/repository/com/awifi/capacity/${appName}/${imageTag}/*.jar ./ && pwd && ls
                       docker login ${dockerRegistryUrl} -u ${dockerHubUser} -p ${dockerHubPassword}
                       docker build -t ${dockerRegistryUrl}/${image}:${imageTag} .
                       docker push ${dockerRegistryUrl}/${image}:${imageTag}
                       """
                }
            }
        }
        stage('运行 Helm 阶段') {
            container('helm-kubectl') {
                echo "5. 运行 Helm 阶段"
                echo "[INFO] 开始运行 Helm 命令"
                if ("true" == "${helmDelete}") {
                    echo "[INFO] Helm 删除应用..."
                    sh "helm delete ${appName} -n ${namespace}"
                    echo "[INFO] Helm 删除应用成功."
                }
                if ("true" == "${helmInstall}") {
                    echo "[INFO] Helm 部署应用..."
                    sh "helm install ${appName} ./${chartDir} --namespace=${namespace} -f ${dockerPath}/values.yaml --set image.repository=192.168.195.2/${image},image.tag=${imageTag},fullnameOverride=${appName},service.type=NodePort"
                    echo "[INFO] Helm 部署应用成功."
                }
                if ("true" == "${helmUpgrade}") {
                    echo "[INFO] Helm 升级应用..."
                    sh "helm upgrade ${appName} ./${chartDir} --namespace=${namespace} -f ${dockerPath}/values.yaml --set image.repository=192.168.195.2/${image},image.tag=${imageTag},fullnameOverride=${appName},service.type=NodePort"
                    echo "[INFO] Helm 升级应用成功"
                }
            }
        }
    }
}
```

上面这段groovy脚本比较简单，我们需要注意的是volumes区域的定义，将容器中的`/root/.m2`目录挂载到`nfs`上是为了给Maven构建添加缓存的，不然每次构建的时候都需要去重新下载依赖，这样就非常慢了；挂载`.kube`目录是为了能够让`kubectl`和`helm`两个工具可以读取到 Kubernetes 集群的连接信息，不然我们是没办法访问到集群的；最后挂载`/var/run/docker.sock`文件是为了能够让我们的docker这个容器获取到Docker Daemon的信息的，因为docker这个镜像里面只有客户端的二进制文件，我们需要使用宿主机的Docker Daemon来构建镜像，当然我们也需要在运行 Slave Pod 的节点上拥有访问集群的文件，然后在每个Stage阶段使用特定需要的容器来进行任务的描述即可，所以这几个volumes都是非常重要的。

```groovy
volumes: [
    nfsVolume(mountPath: '/root/.m2', readOnly: false, serverAddress: '192.168.200.19', serverPath: '/home/k8s-projects/k8s-nfs/m2'),
    hostPathVolume(mountPath: '/home/jenkins/.kube', hostPath: '/root/.kube'),
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]
```

到这里为止，你现在总共有两份文件 Dockerfile 和 Jenkinsfile，把这两份文件放到你在Jenkinsfile脚本中定义的`dockerPath` 目录下，然后提交代码到远程仓库。你还需要将`svnHelm` 目录下的 `values.yaml` 也复制到 `dockerPath` 目录下， `values.yaml` 主要是配置helm安装/更新服务的配置信息。

切换到 Jenkins 页面点击先前定义的流水线任务。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509194818496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

正常可以看到 Jenkins 中的任务构建成功了。

## 8.6 如何配置自定义SpringBoot应用启动参数

在 "8.5 Jenkinsfile 示例" 章节中Jenkinsfile 最后一个阶段 “运行 Helm 阶段” 中，helm 安装或更新服务的命令是

```bash
$ helm upgrade ${appName} ./${chartDir} --namespace=${namespace} -f ${dockerPath}/values.yaml --set \
image.repository=192.168.195.2/${image},image.tag=${imageTag},fullnameOverride=${appName},service.type=NodePort
```

主要关注点是 `values.yaml` : 你可以修改 `container.args` 和 `container.additionalArgs` 来改变Java应用的启动命令参数。

下面是helm中关于容器运行启动的命令配置部分。

`range` 表示循环语句， `container.args`  对应下面的`.Values.container.args` ，`container.additionalArgs` 对应下面的`.Values.container.additionalArgs`

```yaml
...
     containers:
        - name: {{ include "helm-spring-cloud.fullname" . }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command: ["java"]
          args:
            - -Duser.timezone=GMT+8
            - -Djava.security.egd=file:/dev/./urandom
            - -Dserver.port=8080
            - -Dspring.cloud.consul.discovery.prefer-ip-address=false
            - -Dspring.cloud.consul.discovery.hostname={{- include "helm-spring-cloud.fullname" .}}
            {{- range .Values.container.args }}
            - {{ . | quote }}
            {{- end }}
            - -jar
            - /app/app.jar
            {{- range .Values.container.additionalArgs }}
            - {{ . | quote }}
            {{- end }}
...
```

# 参考资源

[jenkins+svn+pipeline+kubernetes部署java应用（一）](https://www.cnblogs.com/xulingjie/p/9916345.html)

[jenkins+svn+pipeline+kubernetes部署java应用（二）](https://www.cnblogs.com/xulingjie/p/9916768.html)

[jenkins+svn+pipeline+kubernetes部署java应用（三）](https://www.cnblogs.com/xulingjie/p/9916904.html)

[玩转Jenkins Pipeline_运维_大宝鱼的博客-CSDN博客](https://blog.csdn.net/diantun00/article/details/81075007)

[基于 kubernetes 的动态 jenkins slave](https://www.qikqiak.com/post/kubernetes-jenkins1/)

[Jenkins Pipeline 部署 Kubernetes 应用(二)](https://www.qikqiak.com/post/kubernetes-jenkins2/)

[Maven的-pl -am -amd参数学习 - hiv - 博客园](https://www.cnblogs.com/hiver/p/7850954.html)

[SVN常用命令之checkout](https://www.cnblogs.com/lxwphp/p/9109685.html)