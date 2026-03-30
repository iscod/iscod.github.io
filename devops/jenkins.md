# Jenkins

## 快速部署 Jenkins（Docker）

```bash
docker run -d -v /var/jenkins_home:/var/jenkins_home -p 8080:8080 --restart=on-failure --name=jenkins jenkins/jenkins:jdk11
```

首次启动后，需要获取初始化管理员密码：

```bash
docker logs jenkins
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 流水线（Pipeline）

Jenkins 流水线 (或简单的带有大写"P"的"Pipeline") 是一套插件，它支持实现和集成 continuous delivery pipelines 到Jenkins。

Jenkins 流水线的定义可以到一个文件中 (Jenkinsfile)保存到源代码的版本控制库中

## 代理（Agent）

agent 指定了整个流水线或特定的部分, 将会在Jenkins环境中执行的位置，这取决于 agent 区域的位置。该部分必须在 pipeline 块的顶层被定义, 但是 stage 级别的使用是可选的。

### 参数

- `none`：不设置全局代理。此时每个 `stage` 都需要显式声明自己的 `agent`（例如 `agent any`）。
- `any`：在任意可用的代理节点上运行流水线或 stage。
- `docker`：在指定的容器镜像内运行流水线或 stage（适合隔离构建环境）。

```jenkins
stage('Build') {
    agent {
        docker {
            image 'golang'
        }
    }
    steps {
        sh 'go version'
    }
}
```

### dockerfile

使用仓库中的 Dockerfile 构建容器作为执行环境。

```jenkins
agent {
    // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
    dockerfile {
        filename 'Dockerfile'
        dir 'build'
        additionalBuildArgs  '--build-arg version=1.0.2'
    }
}
```

## stages

包含一系列一个或多个 stage 指令, stages 部分是流水线描述的大部分"work" 的位置。 建议 stages 至少包含一个 stage 指令用于连续交付过程的每个离散部分,比如构建, 测试, 和部署。

## steps

steps 部分在给定的 stage 指令中执行的定义了一系列的一个或多个steps。


## kubernetes持续集成

![jenkins-kubernetes](https://iscod.github.io/images/jenkins-kubernetes.png)

### 流程说明

1. 用户向Gitlab提交代码，代码中可以包含Dockerfile
2. 将代码提交到远程仓库
3. 用户在发布应用时需要填写git仓库的地址和分支、服务类型、服务名称、资源数量、实例个数、确定后触发Jenkins自动构建
4. Jenkins的CI流水线自动编译代码，并打包docker镜像推送到镜像服务器（如Harbor镜像仓库）
5. Jenkins的CI流水线中包含了自定义脚本，根据已经准备好的kubernetes的YAML模版，将其中的变量替换成用户输入的选项生成应用的Kubernetes YAML配置文件
6. 更新Ingress配置，根据新部署的应用名称，在Ingress的配置文件中增加一条路由信息
7. 更新PowerDNS，向其中插入一条DNS记录。IP地址是边缘节点的IP地址
8. Jenkins调用kubernetes的API，部署应用

### jenkins-X

[jenkins-X](https://jenkins-x.io/zh/)是一个基于 Jenkins 和 Kubernetes 的 CI/CD 平台，旨在解决微服务架构下云原生应用的持续集成和持续交付问题

## 示例 Jenkinsfile

说明：

- 示例里的 `credentialsId`、仓库地址、分支变量等需要按你的环境替换。
- 不建议在公开文档中直接暴露真实 `credentialsId`。

```jenkins
pipeline {
  agent none
  stages {
    stage('Prepare') {
      steps {
        echo 'Prepare...'
      }
    }
    stage('Build') {
      agent {
        docker {
          image 'golang'
        }
      }
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '$COMMIT']], extensions: [], userRemoteConfigs: [[credentialsId: 'YOUR_CREDENTIALS_ID', url: 'git@github.com:YOUR_ORG/YOUR_REPO.git']]])
        sh 'ls -lh'
        sh 'go mod download'
        sh 'go build -a -v -o tao-gin'
      }
    }
    stage('Test') {
        agent any
        steps {
            sh 'ls -lh'
        }
    }
    stage('Deploy') {
        agent any
        steps {
            sh 'if ([ "$ENV" != "" ]); then mv conf/gin.$ENV.conf conf/gin.conf; fi'
            sh 'docker build  -t $JOB_BASE_NAME:$BUILD_NUMBER  .'
        }
    }
  }
}
``` 

## 参考

- [Jenkins Pipeline 官方文档](https://www.jenkins.io/zh/doc/book/pipeline/)
- [jenkins-ci-cd](https://www.bookstack.cn/read/kubernetes-handbook-201910/practice-jenkins-ci-cd.md)
- [Jenkins Kubernetes plugin](https://plugins.jenkins.io/kubernetes/)
- [jenkins-x](https://jenkins-x.io/zh/)