#!/usr/bin/env groovy
import groovy.transform.Field

@Field def job_name = ""

node() {
    // 获取当前 job 名称。也可以按需自定义
    job_name = "${env.JOB_NAME}".replace('%2F', '/').split('/')
    job_name = job_name[0]
    def rootDir = pwd()
    
    echo "job_name: ${job_name}, env.BRANCH_NAME: ${env.BRANCH_NAME}"
    echo "${rootDir}--${job_name}--${JOB_NAME}------${env.JOB_NAME}-----${BRANCH_NAME}-----${env.BRANCH_NAME}"
    // 自定义 workspace
    workspace = "workspace/${job_name}/${env.BRANCH_NAME}"
 
    ws("$workspace") {
        dir("pipeline") {   
            // clone Jenkinsfile 项目
            checkout scmGit(
                branches: [[name: '*/main']], 
                extensions: [], 
                userRemoteConfigs: [[credentialsId: 'git-root', url: 'http://10.255.5.150/cloud-self/devops.git']]
            )
 
            // 根据 job name、构建分支，自动加载对应的 Jenkinsfile
            def check_groovy_file = "Jenkinsfile/${job_name}/${env.BRANCH_NAME}/Jenkinsfile"
            echo "加载 Jenkinsfile 的路径: ${check_groovy_file}"  // 输出路径
            load "${check_groovy_file}"
        }
    }
}