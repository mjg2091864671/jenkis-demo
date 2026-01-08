pipeline {
    agent any

    environment {
        // 目标服务器信息
        SERVER_IP = "192.168.31.61"
        SERVER_USER = "root"
        // 项目相关配置
        PROJECT_NAME = "jenkis-demo"
        JAR_NAME = "${PROJECT_NAME}-0.0.1-SNAPSHOT.jar"
        DEPLOY_DIR = "/opt/apps/${PROJECT_NAME}"
        LOG_FILE = "${DEPLOY_DIR}/${PROJECT_NAME}.log"
        PORT = 8070
        // Git仓库配置
        GIT_URL = "git@github.com:mjg2091864671/jenkis-demo.git"
        GIT_BRANCH = "main"
        GIT_CREDENTIAL_ID = "github-ssh-key"
    }

    // 新增：指定JDK版本（和全局工具配置的JDK名称一致）
    tools {
        jdk 'JDK 17'
        maven 'maven3.9.12'
    }

    stages {
        // 阶段1：拉取代码（无修改）
        stage('拉取代码') {
            steps {
                echo "开始拉取${PROJECT_NAME}代码...仓库：${GIT_URL} 分支：${GIT_BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: "${GIT_URL}",
                        credentialsId: "${GIT_CREDENTIAL_ID}"
                    ]],
                    extensions: [[$class: 'CleanCheckout']]
                ])
            }
        }

        // 阶段2：Maven打包（核心修改：使用指定的JDK和Maven）
        stage('Maven打包') {
            steps {
                echo "开始编译打包${PROJECT_NAME}..."
                script {
                    // 获取Maven路径（也可以直接用tools配置的maven，简化写法）
                    def mavenHome = tool 'maven3.9.12'
                    // 明确指定JDK 17环境，执行mvn编译
                    sh """
                        export JAVA_HOME=${tool 'JDK 17'}
                        export PATH=\$JAVA_HOME/bin:\$PATH
                        ${mavenHome}/bin/mvn clean package -Dmaven.test.skip=true
                    """
                }
                sh "ls -l target/${JAR_NAME}"
            }
        }

        // 阶段3：部署到Linux服务器（无修改）
        stage('部署到Linux服务器') {
            steps {
                script {
                    echo "开始部署${PROJECT_NAME}到${SERVER_IP}..."

                    // 步骤1：远程创建部署目录
                    sh "ssh ${SERVER_USER}@${SERVER_IP} 'mkdir -p ${DEPLOY_DIR}'"

                    // 步骤2：停止旧的SpringBoot服务
                    echo "停止旧的${PROJECT_NAME}服务..."
                    sh """
                        ssh ${SERVER_USER}@${SERVER_IP} '
                            ps -ef | grep ${JAR_NAME} | grep -v grep | awk "{print \$2}" | xargs kill -9 || true
                        '
                    """

                    // 步骤3：上传jar包
                    echo "上传${JAR_NAME}到${SERVER_IP}..."
                    sh "scp target/${JAR_NAME} ${SERVER_USER}@${SERVER_IP}:${DEPLOY_DIR}/"

                    // 步骤4：启动新服务
                    echo "启动${PROJECT_NAME}服务..."
                    sh """
                        ssh ${SERVER_USER}@${SERVER_IP} '
                            cd ${DEPLOY_DIR}
                            nohup java -jar ${JAR_NAME} > ${LOG_FILE} 2>&1 &
                        '
                    """

                    // 步骤5：验证服务启动
                    echo "验证${PROJECT_NAME}服务启动状态..."
                    def maxRetry = 30
                    def retryCount = 0
                    def serviceUp = false
                    while (retryCount < maxRetry && !serviceUp) {
                        sleep 1
                        retryCount++
                        def result = sh(
                            script: "ssh ${SERVER_USER}@${SERVER_IP} 'netstat -tulpn | grep :${PORT} || true'",
                            returnStdout: true
                        ).trim()
                        if (result.contains("LISTEN")) {
                            serviceUp = true
                            echo "${PROJECT_NAME}服务启动成功！端口${PORT}已监听"
                        }
                    }
                    if (!serviceUp) {
                        error "${PROJECT_NAME}服务启动失败！超时${maxRetry}秒，端口${PORT}未监听"
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
    }
}