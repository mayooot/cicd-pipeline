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
