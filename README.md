# cicd-pipeline

## 介绍

使用 GitLab、Jenkins、Kaniko、Harbor、ArgoCD、K8S 实现 GitOps。

以一个[docsify](https://docsify.js.org/#/?id=docsify)项目为例，实现 GitOps。

本项目主要展示Dockerfile、Jenkins Pipeline、Kustomize 等相关文件的编写，不涉及如何配置搭建 Jenkins、Harbor、K8S 等。

## 流程

![图片](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/640.png)

## 环境版本
CI/CD流程中使用到的所有中间件，都部署在 K8S 上。
* Jenkins: 2.414.3
* K8S: v1.24.0
* Harbor: v2.0.4
* ArgoCD: v2.8.4

## 项目结构

> 事实上本项目中的 devops-cd 目录应该作为一个 CD 项目。
> 其余文件作为一个 CI 项目，也就是需要部署的代码文件。
```
.
├── Dockerfile
├── Jenkinsfile		-- Jenkins 流水线
├── README.md
├── devops-cd			-- 持续化部署时，ArgoCD 监听的项目。负责存在 k8s 资源文件。
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── docs					-- docsify 资源文件
    ├── _coverpage.md
    ├── assets
    │   └── _media
    │       ├── favicon.ico
    │       └── icon.svg
    ├── index.html
    └── service
        ├── _sidebar.md
        └── service.md
```

## 准备工作

1. 新建一个 GitLab 项目，将本项目代码提交到该项目。

2. 配置 GitLab Webhook 用于提交代码时触发 Jenkins Pipeline（最下面参考资料 "1"）。

3. Jenkins 安装如下插件：

   * [GitLab Plugin版本1.7.16](https://plugins.jenkins.io/gitlab-plugin)
   * [Kubernetes版本4029.v5712230ccb_f8](https://plugins.jenkins.io/kubernetes)
   * [Pipeline版本596.v8c21c963d92d](https://plugins.jenkins.io/workflow-aggregator)

   并配置Jenkins 可连接 K8S。

4. 创建 CD 项目，将本项目中 `devops-cd` 下的文件提交到该项目。
5. ArgoCD 连接 GitLab CD 项目。

## Jenkins Pipeline

```groovy
pipeline {

    environment {
        harborAddress = 'hub.docker.com' // 这里改为私有 harbor 的域名
        harborRepo = 'docsify' // 仓库名称

        cacheRepo = "${harborAddress}/kanikocache/cache" // 在 harbor 中创建 kanikocache 仓库，用于存储构建缓存

        imageName = "${harborAddress}/${harborRepo}/${JOB_BASE_NAME}:v${currentBuild.number}" // 镜像名称由 域名、仓库、Jenkins 任务名称、当前构建次数拼接而成。每次构建时构建次数会增加，所以不用手动指定镜像版本号。

        app_dir = "${JOB_BASE_NAME}"
        devops_cd_git = "xxx" // 改为 ArgoCD 监听的 GitLab 仓库地址
    }

    agent {
        kubernetes {
            yaml """
            kind: Pod
            metadata:
                name: kaniko
                namespace: jenkins
            spec:
              containers:
              - name: kaniko
                image: ghostwritten/kaniko-project-executor:debug
                imagePullPolicy: Always
                command:
                - /busybox/cat
                tty: true
                volumeMounts:
                - name: jenkins-docker-cfg
                  mountPath: /kaniko/.docker
              - name: kustomize
                image: registry.cn-hangzhou.aliyuncs.com/rookieops/kustomize:v3.8.1
                command:
                - cat
                tty: true
              volumes:
              - name: jenkins-docker-cfg
                projected:
                  sources:
                  - secret:
                      name: harbor-regcred // 使用 kubectl create secret docker-registry NAME 的方式，创建名为 harbor-regcred 的 secret
                      items:
                      - key: .dockerconfigjson
                        path: config.json

            """
        }
    }

    stages {
        stage('Pull code from the specified branch') {
            steps {
                git credentialsId: 'xxx', url: 'ci_url' // credentialsId 为 GitLab 账号凭证、url 为 CI 项目的地址
            }
        }

        stage('Kaniko Build and Push Docker Image') {
            steps {
                container(name: 'kaniko', shell: '/busybox/sh') {
                    sh '''#!/busybox/sh
                    echo `pwd`
                    /kaniko/executor -f `pwd`/Dockerfile -c `pwd` --cache=true --cache-repo=${cacheRepo} --destination=${imageName}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding',
                credentialsId: 'xxx',
                usernameVariable: 'DEVOPS_USER',
                passwordVariable: 'DEVOPS_PASSWORD']]) {
                    container('kustomize') {
                        script {
                            sh """
                            git remote set-url origin http://${DEVOPS_USER}:${DEVOPS_PASSWORD}@${devops_cd_git}
                            git config --global user.name "Administrator"
                            git config --global user.email "xxx@163.com"
                            git clone https://${DEVOPS_USER}:${DEVOPS_PASSWORD}@${devops_cd_git} /opt/devops-cd
                            cd /opt/devops-cd
                            git pull
                            cd /opt/devops-cd/${app_dir}
                            kustomize edit set image ${imageName}
                            git commit -am 'image update'
                            git push origin master
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        // 最后结果通知企业微信机器人
        success {
            qyWechatNotification mentionedId: '', mentionedMobile: '', moreInfo: '', webhookUrl: 'url'
        }

        failure {
            qyWechatNotification mentionedId: '', mentionedMobile: '', moreInfo: '', webhookUrl: 'url'
        }
    }
}

```



## 演示
当提交代码时，自动触发 Jenkins Pipeline。
![image-20231107165511035](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107165511035.png)



构建镜像后，分发到 Harbor 仓库，然后提交修改到 CD 项目中。

![image-20231107165814457](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107165814457.png)

ArgoCD 重新部署项目。
![image-20231107165740019](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107165740019.png)

企业微信机器人通知。

![image-20231107170106466](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107170106466.png)

项目更新
![image-20231107152900800](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107152900800.png)
![image-20231107152922679](https://bertram-li-bucket.oss-cn-beijing.aliyuncs.com/markdown-img/image-20231107152922679.png)

## 参考资料

1. [ GitLab 配置Webhook](https://zhuanlan.zhihu.com/p/540353361)
3. [Jenkins连接 K8s](https://juejin.cn/post/6963466680613896206)
4. [ArgoCD 绑定 GitLab 项目](https://www.qikqiak.com/post/gitlab-ci-argo-cd-gitops/)
5. [基于Jenkins和Argocd实现CI/CD](https://mp.weixin.qq.com/s?__biz=MzIyMDY2MTE3Mw==&amp;mid=2247486319&amp;idx=1&amp;sn=cd38bba1ca271fed1251bf15a72511d1&amp;chksm=97c9dfb5a0be56a3b5747e9af558b42bbdb767a4e045c9a98820cdb99df7445a3e8eba3a047e&amp;scene=21#wechat_redirect)
