#!/usr/bin/env groovy

pipeline {
    agent {
        node {
            label "master"
        }
    }

    stages {
        stage('1.拉取代码') {
            steps {
                script {
                    def git_address = "http://192.168.0.153:8090/lins/teset-server.git"
                    def git_auth = "10a4d4f3-1977-486c-945f-31cfcd8c04db"
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
                    export GOPATH=/opt/GOPATH
                    export PATH=\$PATH:\$GOPATH/bin
                    go build -o training-ip-demo main.go
                    cp training-ip-demo /home/training-ip-management/training-ip-demo
                """
            }
        }

        stage('3.部署程序') {
            steps {
                script {
                    def deployConfig = [
                        dev: [host: "192.168.1.10", path: "/home/dev/training-ip-demo", sshCredentialId: "dev-ssh-key"],
                        main: [host: "192.168.0.132", path: "/home/test/training-ip-demo", sshCredentialId: "main-ssh-key"],
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
                            ssh user@${config.host} "
                                OLD_PID=\$(ps -ef | grep training-ip-demo | grep -v grep | grep -v jenkins | awk '{print \$2}')
                                if [ -n \"\$OLD_PID\" ]; then
                                    echo \"找到旧进程 ID: \$OLD_PID\"
                                    sudo kill -9 \$OLD_PID
                                else
                                    echo \"未找到旧进程\"
                                fi
                            "
                        """

                        // 部署新程序
                        sh """
                            scp /home/training-ip-management/training-ip-demo user@${config.host}:${config.path}/
                            ssh user@${config.host} "
                                cd ${config.path} && 
                                JENKINS_NODE_COOKIE=dontKillMe nohup ./training-ip-demo > /dev/null 2>&1 &
                            "
                        """
                    }
                }
            }
        }
    }
}
