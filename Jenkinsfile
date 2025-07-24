#!/usr/bin/env groovy

pipeline {
    agent {
        node {
            label "master"
        }
    }

    tools {
        go 'go1.23.0'  // 使用 Go 插件预装的 Go 版本（需在 Jenkins 全局工具配置中定义）
    }

    environment {
        PATH = "${env.GOPATH}:${env.PATH}"
    }

    stages {
        stage('1.拉取代码') {
            steps {
                script {
                    def git_address = "http://192.168.0.153:8090/lins/teset-server.git"
                    def git_auth = "7cb729e5-a40c-445b-b82d-bac1fb6df325"
                    def git_branch = env.BRANCH_NAME
                    
                    git(
                        branch: git_branch, 
                        credentialsId: git_auth, 
                        url: git_address
                    )
                }
            }
        }

        stage('2.编译程序') {
            steps {
                sh """
                    #!/bin/bash
                    # 不需要手动设置 GOPATH 和 PATH，Go 插件已自动配置
                    cd ${env.WORKSPACE}
                    export GOPROXY=https://goproxy.cn,direct
                    go build -o teset-server main.go
                    cp teset-server /var/jenkins_home/teset-server
                """
            }
        }

        stage('3.部署程序') {
            steps {
                script {
                    def deployConfig = [
                        dev: [host: "192.168.1.10", path: "/home/dev/training-ip-demo", sshCredentialId: "dev-ssh-key"],
                        main: [host: "192.168.0.210", path: "/home/main/teset-server", sshCredentialId: "main-ssh-key"],
                        prod: [host: "192.168.1.30", path: "/home/prod/training-ip-demo", sshCredentialId: "prod-ssh-key"]
                    ]

                    def targetBranch = git_branch
                    def config = deployConfig[targetBranch]

                    if (!config) {
                        error("未配置该分支的部署信息: ${targetBranch}")
                    }

                    sshagent([config.sshCredentialId]) {
                        // 杀死目标服务器上的旧进程
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${config.host} "
                                echo '开始查找并终止旧进程...'
                                OLD_PID=\$(ps -ef | grep teset-server | grep -v grep | grep -v jenkins | awk '{print \$2}')
                                if [ -n \"\$OLD_PID\" ]; then
                                    echo \"找到旧进程 ID: \$OLD_PID，正在终止...\"
                                    sudo kill -9 \$OLD_PID || echo \"终止旧进程失败\"
                                else
                                    echo '未找到旧进程'
                                fi
                            "
                        """

                        // 部署新程序
                        sh """
                            echo '开始部署新程序...'
                            scp /var/jenkins_home/teset-server root@${config.host}:${config.path}/
                            ssh -o StrictHostKeyChecking=no root@${config.host} "
                                echo '创建部署目录...'
                                mkdir -p ${config.path}
                                echo '启动新程序...'
                                cd ${config.path} && 
                                JENKINS_NODE_COOKIE=dontKillMe nohup ./teset-server > /dev/null 2>&1 &
                            "
                        """
                    }
                }
            }
        }
    }
}
