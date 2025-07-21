#!/usr/bin/env groovy

def git_address = "git@101.132.137.193:trainingipmanagement/training-ip-demo.git"
def git_auth = "10a4d4f3-1977-486c-945f-31cfcd8c04db"
def git_branch = "dev"


pipeline {
    agent { node { label "master"}}

    stages {
        stage('1.拉取代码'){
            steps {
                git  branch: "${git_branch}", credentialsId: "${git_auth}", url: "${git_address}"
            }
        }

        stage('3.编译程序'){
            steps {
                sh """  
                    export GOPATH=/opt/GOPATH
                    export PATH=$PATH:\$GOPATH/bin
                    go build -o training-ip-demo main.go
                    cp training-ip-demo /home/training-ip-management/training-ip-demo       
                """   
            }      
        }  
        
        stage('4.部署程序'){
            steps {
                script {
                    // 根据分支选择不同的部署配置（含 SSH 凭据 ID）
                    def deployConfig = [
                        dev: [host: "192.168.1.10", path: "/home/dev/training-ip-demo", sshCredentialId: "dev-ssh-key"],
                        test: [host: "192.168.1.20", path: "/home/test/training-ip-demo", sshCredentialId: "test-ssh-key"],
                        prod: [host: "192.168.1.30", path: "/home/prod/training-ip-demo", sshCredentialId: "prod-ssh-key"]
                    ]

                    def targetBranch = "${git_branch}"
                    def config = deployConfig[targetBranch]

                    if (!config) {
                        error("未配置该分支的部署信息: ${targetBranch}")
                    }

                    // 使用 sshagent 加载 SSH 密钥并执行部署
                    sshagent(credentials: [config.sshCredentialId]) {
                        // 杀死目标服务器上的旧进程
                        sh """
                            ssh user@${config.host} '
                                # 查找旧进程
                                OLD_PID=\$(ps -ef | grep training-ip-demo | grep -v grep | grep -v jenkins | awk "{print \\$2}")
                                if [ -n "\$OLD_PID" ]; then
                                    echo "找到旧进程 ID: \$OLD_PID"
                                    sudo kill -9 \$OLD_PID
                                else
                                    echo "未找到旧进程"
                                fi
                            '
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

    post {
        always{
            script{
                println("流水线结束后，经常做的事情")
            }
        }
            
        success{
            script{
                println("流水线成功后，要做的事情")
            }
            
        }
        failure{
            script{
                println("流水线失败后，要做的事情")
            }
        }
            
        aborted{
            script{
                println("流水线取消后，要做的事情")
            }
            
        }
    }
}