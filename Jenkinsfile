pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: '部署的分支；如仓库使用 master，请改为 master')
    }

    environment {
        GIT_URL = 'git@github.com:zxsking/demo.git'
        DEPLOY_USER = 'root'
        APP_NAME = 'demo'
        DEPLOY_HOST = '81.71.38.91'
        DEPLOY_DIR = '/opt/apps/demo'
        SERVICE_NAME = 'demo'
    }

    tools {
        jdk 'JDK 17'
        maven 'Maven 3.9.5'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('拉取代码') {
            steps {
                git branch: "${params.GIT_BRANCH}",
                    credentialsId: 'github-ssh-key',
                    url: "${GIT_URL}"
            }
        }

        stage('构建与测试') {
            steps {
                sh '''
                    mvn clean verify
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('准备部署包') {
            steps {
                sh '''
                    JAR_PATH=$(find target -maxdepth 1 -type f -name '*.jar' ! -name 'original-*.jar' | head -n 1)

                    if [ -z "$JAR_PATH" ]; then
                        echo "未找到构建后的 Jar 文件"
                        exit 1
                    fi

                    cp "$JAR_PATH" app.jar
                '''
                archiveArtifacts artifacts: 'app.jar', fingerprint: true
            }
        }

      stage('部署') {
            steps {
                sh '''
                    set -eu
        
                    SSH_TARGET="${DEPLOY_USER}@${DEPLOY_HOST}"
        
                    # 创建目录、停止旧进程、备份旧包
                    ssh "${SSH_TARGET}" "
                        set -eu
                        mkdir -p '${DEPLOY_DIR}/backup' '${DEPLOY_DIR}/logs'
        
                        if [ -f '${DEPLOY_DIR}/app.pid' ]; then
                            PID=\\$(cat '${DEPLOY_DIR}/app.pid')
        
                            if kill -0 \\$PID 2>/dev/null; then
                                kill \\$PID
                                sleep 5
                            fi
        
                            rm -f '${DEPLOY_DIR}/app.pid'
                        fi
        
                        if [ -f '${DEPLOY_DIR}/app.jar' ]; then
                            cp '${DEPLOY_DIR}/app.jar' \
                               '${DEPLOY_DIR}/backup/app-'\\$(date +%Y%m%d%H%M%S)'.jar'
                        fi
                    "
        
                    # 上传完成后再替换，避免文件上传中覆盖旧程序
                    scp app.jar "${SSH_TARGET}:${DEPLOY_DIR}/app.jar.new"
        
                    # 启动新版本并保存进程 ID
                    ssh "${SSH_TARGET}" "
                        set -eu
                        mv '${DEPLOY_DIR}/app.jar.new' '${DEPLOY_DIR}/app.jar'
        
                        nohup java -jar '${DEPLOY_DIR}/app.jar' \
                            --spring.profiles.active=prod \
                            > '${DEPLOY_DIR}/logs/app.log' 2>&1 < /dev/null &
        
                        echo \\$! > '${DEPLOY_DIR}/app.pid'
                        sleep 5
        
                        PID=\\$(cat '${DEPLOY_DIR}/app.pid')
                        kill -0 \\$PID
                    "
                '''
            }
        }
        stage('健康检查') {
            steps {
                sh '''
                    set -eu
        
                    sleep 10
                    curl --fail --silent --show-error \
                        --retry 3 --retry-delay 5 \
                        "http://${DEPLOY_HOST}:8081/"
                '''
            }
        }
    }

    post {
        success {
            echo 'Spring Boot 项目部署成功。'
        }
        failure {
            echo '流水线失败，请检查 Jenkins 控制台日志。'
        }
        always {
            cleanWs()
        }
    }
    
}