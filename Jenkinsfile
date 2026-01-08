/**
 * SpringBoot项目Jenkins自动部署流水线
 * 流程：拉取代码 → Maven打包 → 部署到Linux服务器（优雅启停+版本备份+端口验证）
 */
pipeline {
    agent any
    // 1. 新增：定义参数化构建参数（核心修改点1）
    parameters {
        choice(
            name: 'BRANCH_TO_DEPLOY',  // 参数名，后续引用使用 params.BRANCH_TO_DEPLOY
            choices: [
                'main',               // 默认分支
                'dev',                // 开发分支
                'test',               // 测试分支
                'release'             // 发布分支
            ],
            description: '请选择要部署的Git分支'  // 参数描述
        )
        // 可选：如果需要支持自定义分支名，可替换为string参数（二选一）
        // string(
        //     name: 'BRANCH_TO_DEPLOY',
        //     defaultValue: 'main',
        //     description: '请输入要部署的Git分支名（如main/dev/test）'
        // )
    }

    options {
        timeout(time: 30, unit: 'MINUTES')  // 全局超时30分钟
        buildDiscarder(logRotator(numToKeepStr: '10'))  // 保留最近10次构建记录
    }

    environment {
        // 服务器凭据（从Jenkins凭据库读取）
        SERVER_CRED = credentials('server-credential')
        SERVER_IP = "192.168.31.61"
        SERVER_USER = "${SERVER_CRED_USR}"

        // 项目配置
        PROJECT_NAME = "jenkins-demo"
        JAR_NAME = "${PROJECT_NAME}-0.0.1-SNAPSHOT.jar"
        DEPLOY_DIR = "/opt/apps/${PROJECT_NAME}"
        BACKUP_DIR = "${DEPLOY_DIR}/backup"
        LOG_FILE = "${DEPLOY_DIR}/${PROJECT_NAME}.log"
        PORT = 8070

        // Git配置
        GIT_URL = "git@github.com:mjg2091864671/jenkins-demo.git"
        GIT_BRANCH = "main"
        GIT_CREDENTIAL_ID = "github-ssh-key"

        // JVM启动参数
        JVM_OPTS = "-Xms512m -Xmx1024m -XX:+UseG1GC -Dspring.profiles.active=prod"
    }

    tools {
        jdk 'JDK 17'
        maven 'maven3.9.12'
    }

    stages {
        stage('拉取代码') {
            options { timeout(time: 5, unit: 'MINUTES') }
            steps {
                echo "开始拉取${PROJECT_NAME}代码...仓库：${GIT_URL} 分支：${GIT_BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${GIT_BRANCH}"]],
                    userRemoteConfigs: [[url: "${GIT_URL}", credentialsId: "${GIT_CREDENTIAL_ID}"]],
                    extensions: [[$class: 'CleanCheckout']]  // 拉取前清理工作区
                ])
            }
        }

        stage('Maven打包') {
            options { timeout(time: 10, unit: 'MINUTES') }
            steps {
                echo "开始编译打包${PROJECT_NAME}..."
                withMaven(maven: 'maven3.9.12', jdk: 'JDK 17') {
                    sh "mvn clean package -Dmaven.test.skip=true -U"
                }

                // 校验打包结果
                script {
                    def jarPath = "target/${JAR_NAME}"
                    if (!fileExists(jarPath)) {
                        error "打包失败：${jarPath} 文件不存在！"
                    }
                    sh "ls -l ${jarPath}"
                }
            }
        }

        stage('创建部署目录') {
            options { timeout(time: 2, unit: 'MINUTES') }
            steps {
                script {
                    def remoteExec = { String cmd ->
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }
                    echo "创建部署/备份目录..."
                    remoteExec("mkdir -p ${DEPLOY_DIR} ${BACKUP_DIR}")
                }
            }
        }

        stage('停止旧服务') {
            options { timeout(time: 3, unit: 'MINUTES') }
            steps {
                script {
                    def remoteExec = { String cmd ->
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }
                    echo "停止旧的${PROJECT_NAME}服务..."
                    remoteExec("""
                        lsof -i:${PORT} -t | xargs kill -9 2>/dev/null || echo "端口${PORT}无占用进程"
                        PID=\$(ps -ef | grep ${JAR_NAME} | grep -v grep | awk '{print \$2}')
                        if [ -n "\$PID" ]; then
                            kill \$PID && sleep 5
                            if ps -p \$PID > /dev/null; then
                                kill -9 \$PID && echo "强制杀死进程\$PID"
                            fi
                        fi
                    """)
                }
            }
        }

        stage('备份旧版本') {
            options { timeout(time: 3, unit: 'MINUTES') }
            steps {
                script {
                    def remoteExec = { String cmd ->
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }
                    echo "备份旧版本JAR包..."
                    remoteExec("""
                        if [ -f ${DEPLOY_DIR}/${JAR_NAME} ]; then
                            cp ${DEPLOY_DIR}/${JAR_NAME} ${BACKUP_DIR}/${JAR_NAME}.\$(date +%Y%m%d%H%M%S)
                            echo "旧版本已备份到${BACKUP_DIR}"
                        fi
                    """)
                }
            }
        }

        stage('上传JAR包') {
            options { timeout(time: 5, unit: 'MINUTES') }
            steps {
                echo "上传${JAR_NAME}到${SERVER_IP}..."
                sh "scp -o StrictHostKeyChecking=no target/${JAR_NAME} ${SERVER_USER}@${SERVER_IP}:${DEPLOY_DIR}/"
            }
        }

        stage('启动新服务') {
            options { timeout(time: 5, unit: 'MINUTES') }
            steps {
                script {
                    def remoteExec = { String cmd ->
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }
                    echo "启动${PROJECT_NAME}服务..."
                    remoteExec("""
                        cd ${DEPLOY_DIR}
                        [ -f ${LOG_FILE} ] && mv ${LOG_FILE} ${LOG_FILE}.\$(date +%Y%m%d%H%M%S)
                        nohup java ${JVM_OPTS} -jar ${JAR_NAME} > ${LOG_FILE} 2>&1 &
                    """)
                }
            }
        }

        stage('验证服务启动') {
            options { timeout(time: 5, unit: 'MINUTES') }
            steps {
                script {
                    def remoteExec = { String cmd ->
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }
                    echo "验证${PROJECT_NAME}服务启动状态..."
                    def maxRetry = 30
                    def retryCount = 0
                    def serviceUp = false
                    while (retryCount < maxRetry && !serviceUp) {
                        sleep 1
                        retryCount++
                        def result = sh(
                            script: "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} 'netstat -tulpn | grep :${PORT} || true'",
                            returnStdout: true
                        ).trim()
                        if (result.contains("LISTEN")) {
                            serviceUp = true
                            echo "${PROJECT_NAME}服务启动成功！端口${PORT}已监听"
                        }
                    }
                    if (!serviceUp) {
                        echo "部署失败，输出最新日志："
                        remoteExec("tail -20 ${LOG_FILE}")
                        error "${PROJECT_NAME}服务启动失败！超时${maxRetry}秒"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "===== ${PROJECT_NAME}部署成功！====="
            echo "部署服务器：${SERVER_IP}"
            echo "部署目录：${DEPLOY_DIR}"
            echo "日志文件：${LOG_FILE}"
        }
        failure {
            echo "===== ${PROJECT_NAME}部署失败！====="
        }
        always {
            echo "清理本地构建产物..."
            sh "rm -rf target/* || true"
        }
    }
}