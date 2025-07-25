pipeline {
    agent {
        node {
            label "master"
        }
    }

    tools {
        go 'go1.23.0'
    }

    environment {
        PATH = "${env.GOPATH}:${env.PATH}"
        PROJECT_NAME = sh(script: """
            echo ${env.GIT_URL} | awk -F'/' '{print $NF}' | sed 's/\\..*//'
        """, returnStdout: true).trim()
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
                    cd ${env.WORKSPACE}
                    export GOPROXY=https://goproxy.cn,direct
                    go build -o ${env.PROJECT_NAME} main.go
                    cp ${env.PROJECT_NAME} /var/jenkins_home/${env.PROJECT_NAME}
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
                                OLD_PID=\$(ps -ef | grep ${env.PROJECT_NAME} | grep -v grep | grep -v jenkins | awk '{print \$2}')
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
                            scp /var/jenkins_home/${env.PROJECT_NAME} root@${config.host}:${config.path}/
                            ssh -o StrictHostKeyChecking=no root@${config.host} "
                                echo '创建部署目录...'
                                mkdir -p ${config.path}
                                echo '启动新程序...'
                                cd ${config.path} && 
                                JENKINS_NODE_COOKIE=dontKillMe nohup ./${env.PROJECT_NAME} > /dev/null 2>&1 &
                            "
                        "
                    }
                }
            }
        }
    }

    post {
        success {
            dingTalk (
                robot: 'dev-robot',  // 引用系统配置的机器人 ID
                message: """
{
    "msgtype": "markdown",
    "markdown": {
        "title": "构建成功",
        "text": "### 🎉 构建成功\n\n" +
                "项目名称: `\${env.PROJECT_NAME}`\n" +
                "分支: `${env.BRANCH_NAME}`\n" +
                "构建结果: 成功 ✅\n" +
                "构建链接: [点击查看](${env.BUILD_URL})\n\n" +
                "> 自动部署完成，请检查目标服务器状态。"
    }
}
                """
            )
        }

        failure {
            dingTalk (
                robot: 'dev-robot',
                message: """
{
    "msgtype": "markdown",
    "markdown": {
        "title": "构建失败",
        "text": "### ❌ 构建失败\n\n" +
                "项目名称: `\${env.PROJECT_NAME}`\n" +
                "分支: `${env.BRANCH_NAME}`\n" +
                "构建结果: 失败 ❌\n" +
                "构建链接: [点击查看](${env.BUILD_URL})\n\n" +
                "> 请检查 Jenkins 日志以获取详细错误信息。"
    }
}
                """
            )
        }
    }
}
