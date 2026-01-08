/**
 * 【SpringBoot项目Jenkins自动部署流水线】
 * 核心功能：拉取Git代码 → Maven打包 → 部署到Linux服务器（优雅启停+版本备份+端口验证）
 * 适用场景：新手学习Jenkins部署SpringBoot，兼顾生产环境基础规范
 * 关键优化：凭据管理、优雅启停、版本备份、超时控制、日志轮转
 */
pipeline {
    // agent any：表示使用任意可用的Jenkins节点（从节点/主节点）执行流水线
    // 新手提示：如果有专用构建节点，可以指定agent { label 'build-node' }
    agent any

    // 流水线全局选项配置（作用于整个流水线生命周期）
    options {
        // 全局超时：防止某个阶段卡死导致流水线一直挂起，30分钟未完成则终止
        timeout(time: 30, unit: 'MINUTES')
        // 构建日志清理：只保留最近10次构建记录，避免Jenkins磁盘占满
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // 环境变量定义（统一管理配置，便于维护和修改）
    environment {
        /************************ 服务器认证配置（安全规范） ************************/
        // 从Jenkins凭据库读取服务器认证信息（推荐做法，避免明文硬编码）
        // 新手操作：Jenkins → 凭据 → 系统 → 全局凭据 → 添加凭据（类型：Username with password，ID填server-credential）
        SERVER_CRED = credentials('server-credential') 
        // 目标服务器IP（固定配置）
        SERVER_IP = "192.168.31.61"
        // 从凭据中提取用户名（避免明文写root）
        SERVER_USER = "${SERVER_CRED_USR}" 

        /************************ 项目核心配置（可根据项目修改） ************************/
        // 项目名称（统一命名，便于识别）
        PROJECT_NAME = "jenkins-demo"
        // 打包后的JAR文件名（需和pom.xml中finalName一致）
        JAR_NAME = "${PROJECT_NAME}-0.0.1-SNAPSHOT.jar"
        // 服务器部署目录
        DEPLOY_DIR = "/opt/apps/${PROJECT_NAME}"
        // 版本备份目录（部署前备份旧版本，方便回滚）
        BACKUP_DIR = "${DEPLOY_DIR}/backup"
        // 应用日志文件路径
        LOG_FILE = "${DEPLOY_DIR}/${PROJECT_NAME}.log"
        // 应用监听端口（需和SpringBoot配置文件一致）
        PORT = 8070

        /************************ Git仓库配置 ************************/
        // Git仓库地址（SSH方式，需配置Jenkins的SSH凭据）
        GIT_URL = "git@github.com:mjg2091864671/jenkins-demo.git"
        // 要拉取的分支
        GIT_BRANCH = "main"
        // Jenkins中配置的Git SSH凭据ID（新手操作：添加SSH密钥凭据，ID填github-ssh-key）
        GIT_CREDENTIAL_ID = "github-ssh-key"

        /************************ JVM启动参数（生产环境必备） ************************/
        // Xms：初始堆内存；Xmx：最大堆内存；UseG1GC：垃圾收集器；spring.profiles.active：指定生产环境配置
        JVM_OPTS = "-Xms512m -Xmx1024m -XX:+UseG1GC -Dspring.profiles.active=prod"
    }

    // 工具配置（关联Jenkins全局工具配置，自动注入环境变量）
    // 新手操作：Jenkins → 全局工具配置 → 配置JDK和Maven，名称要和这里一致（JDK 17、maven3.9.12）
    tools {
        jdk 'JDK 17'    // 指定构建使用的JDK版本（需和SpringBoot项目的JDK一致）
        maven 'maven3.9.12' // 指定构建使用的Maven版本
    }

    // 流水线阶段（按部署流程分阶段执行，便于排查问题）
    stages {
        // 阶段1：拉取Git代码
        stage('拉取代码') {
            // 阶段级超时：该阶段5分钟未完成则终止（精细化控制）
            options { timeout(time: 5, unit: 'MINUTES') }
            steps {
                echo "开始拉取${PROJECT_NAME}代码...仓库：${GIT_URL} 分支：${GIT_BRANCH}"
                // Jenkins Git拉取代码的标准语法
                checkout([
                    $class: 'GitSCM',          // 指定使用GitSCM插件
                    branches: [[name: "*/${GIT_BRANCH}"]], // 拉取远程分支（*/表示远程分支）
                    userRemoteConfigs: [[
                        url: "${GIT_URL}",      // Git仓库地址
                        credentialsId: "${GIT_CREDENTIAL_ID}" // Git认证凭据
                    ]],
                    // CleanCheckout：拉取前清理工作区（避免旧代码干扰，新手推荐开启）
                    extensions: [[$class: 'CleanCheckout']]
                ])
            }
        }

        // 阶段2：Maven打包（编译源码生成可执行JAR包）
        stage('Maven打包') {
            options { timeout(time: 10, unit: 'MINUTES') }
            steps {
                echo "开始编译打包${PROJECT_NAME}..."
                // withMaven：自动关联配置的JDK和Maven，无需手动配置环境变量（新手友好）
                withMaven(maven: 'maven3.9.12', jdk: 'JDK 17') {
                    // mvn命令说明：
                    // clean：清理之前的构建产物
                    // package：打包生成JAR包
                    // -Dmaven.test.skip=true：跳过单元测试（加快打包速度，生产环境建议保留测试）
                    // -U：强制更新快照依赖（避免依赖缓存导致的问题）
                    sh "mvn clean package -Dmaven.test.skip=true -U"
                }

                // 校验打包结果（关键：避免打包失败但流水线继续执行的问题）
                script {
                    def jarPath = "target/${JAR_NAME}"
                    // fileExists：检查文件是否存在
                    if (!fileExists(jarPath)) {
                        // error：抛出异常终止流水线，并输出错误信息
                        error "打包失败：${jarPath} 文件不存在！"
                    }
                    // 列出JAR包信息，便于排查（日志中可看到文件大小、修改时间）
                    sh "ls -l ${jarPath}"
                }
            }
        }

        // 阶段3：部署到Linux服务器（核心部署逻辑）
        stage('部署到Linux服务器') {
            options { timeout(time: 15, unit: 'MINUTES') }
            steps {
                script {
                    echo "开始部署${PROJECT_NAME}到${SERVER_IP}..."
                    
                    /************************ 封装远程执行函数（简化代码） ************************/
                    // 定义remoteExec函数：封装SSH远程执行命令的逻辑，避免重复写SSH命令
                    // 参数cmd：要在远程服务器执行的命令
                    def remoteExec = { String cmd ->
                        // ssh命令参数说明：
                        // -o StrictHostKeyChecking=no：关闭主机密钥检查（避免首次连接时手动确认，导致流水线阻塞）
                        // ${SERVER_USER}@${SERVER_IP}：远程服务器账号+IP
                        // '${cmd}'：要执行的远程命令
                        sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} '${cmd}'"
                    }

                    /************************ 步骤1：创建部署/备份目录 ************************/
                    // mkdir -p：递归创建目录，目录已存在则不报错（避免目录不存在导致部署失败）
                    remoteExec("mkdir -p ${DEPLOY_DIR} ${BACKUP_DIR}")

                    /************************ 步骤2：优雅停止旧服务（核心优化） ************************/
                    echo "停止旧的${PROJECT_NAME}服务..."
                    remoteExec("""
                        # 第一步：按端口查找并杀死占用进程（核心修复）
                       # lsof -i:端口：查找占用端口的进程；-t：仅输出PID；xargs kill -9：强制杀死
                       lsof -i:${PORT} -t | xargs kill -9 2>/dev/null || echo "端口${PORT}无占用进程"

                       # 第二步：按JAR包名兜底杀死进程（兼容lsof未安装的情况）
                       PID=\$(ps -ef | grep ${JAR_NAME} | grep -v grep | awk '{print \$2}')
                       if [ -n "\$PID" ]; then
                           kill \$PID && sleep 5
                           if ps -p \$PID > /dev/null; then
                               kill -9 \$PID && echo "强制杀死JAR进程\$PID"
                           fi
                       fi
                    """)

                    /************************ 步骤3：备份旧版本（支持回滚） ************************/
                    echo "备份旧版本jar包..."
                    remoteExec("""
                        # 检查旧JAR包是否存在
                        if [ -f ${DEPLOY_DIR}/${JAR_NAME} ]; then
                            # 备份格式：JAR名.时间戳（便于区分不同版本）
                            cp ${DEPLOY_DIR}/${JAR_NAME} ${BACKUP_DIR}/${JAR_NAME}.\$(date +%Y%m%d%H%M%S)
                            echo "旧版本已备份到${BACKUP_DIR}"
                        fi
                    """)

                    /************************ 步骤4：上传JAR包到服务器 ************************/
                    echo "上传${JAR_NAME}到${SERVER_IP}..."
                    // scp命令：本地文件上传到远程服务器
                    // -o StrictHostKeyChecking=no：同上，避免首次连接阻塞
                    sh "scp -o StrictHostKeyChecking=no target/${JAR_NAME} ${SERVER_USER}@${SERVER_IP}:${DEPLOY_DIR}/"

                    /************************ 步骤5：启动新服务 ************************/
                    echo "启动${PROJECT_NAME}服务..."
                    remoteExec("""
                        cd ${DEPLOY_DIR}  # 进入部署目录
                        # 日志轮转：将旧日志重命名（避免单个日志文件过大，便于问题排查）
                        [ -f ${LOG_FILE} ] && mv ${LOG_FILE} ${LOG_FILE}.\$(date +%Y%m%d%H%M%S)
                        # 启动命令说明：
                        # nohup：脱离终端运行（关闭SSH后应用不停止）
                        # java ${JVM_OPTS}：指定JVM参数启动应用
                        # -jar ${JAR_NAME}：运行JAR包
                        # > ${LOG_FILE} 2>&1：将标准输出和错误输出重定向到日志文件
                        # &：后台运行
                        nohup java ${JVM_OPTS} -jar ${JAR_NAME} > ${LOG_FILE} 2>&1 &
                    """)

                    /************************ 步骤6：验证服务启动（关键） ************************/
                    echo "验证${PROJECT_NAME}服务启动状态..."
                    def maxRetry = 30    // 最大重试次数（30秒）
                    def retryCount = 0   // 当前重试次数
                    def serviceUp = false // 服务是否启动成功标记
                    // 循环检查：每隔1秒检查一次端口，直到超时或端口监听
                    while (retryCount < maxRetry && !serviceUp) {
                        sleep 1           // 暂停1秒
                        retryCount++      // 重试次数+1
                        // 检查端口监听状态：
                        // netstat -tulpn：列出所有监听的端口和对应进程
                        // grep :${PORT}：筛选指定端口
                        // || true：避免命令执行失败导致流水线终止
                        def result = sh(
                            script: "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_IP} 'netstat -tulpn | grep :${PORT} || true'",
                            returnStdout: true // 返回命令输出（而非执行状态）
                        ).trim() // 去除首尾空格
                        // 检查输出是否包含LISTEN（表示端口已监听，服务启动成功）
                        if (result.contains("LISTEN")) {
                            serviceUp = true
                            echo "${PROJECT_NAME}服务启动成功！端口${PORT}已监听"
                        }
                    }
                    // 超时未启动则终止流水线，并输出日志便于排查
                    if (!serviceUp) {
                        echo "部署失败，输出最新日志："
                        remoteExec("tail -20 ${LOG_FILE}") // 输出最后20行日志
                        error "${PROJECT_NAME}服务启动失败！超时${maxRetry}秒，端口${PORT}未监听"
                    }
                }
            }
        }
    }

    // 流水线后置操作（流水线执行完成后触发，无论成功/失败）
    post {
        // 成功后置：流水线全部阶段执行成功时触发
        success {
            echo "===== ${PROJECT_NAME}部署成功！====="
            echo "部署服务器：${SERVER_IP}"
            echo "部署目录：${DEPLOY_DIR}"
            echo "日志文件：${LOG_FILE}"
        }
        // 失败后置：流水线任意阶段失败时触发
        failure {
            echo "===== ${PROJECT_NAME}部署失败！====="
            // 新手拓展：可添加邮件/钉钉通知，示例：
            // emailext to: 'your-email@xxx.com', subject: '部署失败', body: '项目部署失败，请排查！'
        }
        // 始终执行：无论成功/失败都触发
        always {
            echo "清理本地构建产物..."
            // 清理target目录（避免Jenkins工作区磁盘占满）
            sh "rm -rf target/* || true"
        }
    }
}