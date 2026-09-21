pipeline {
    agent any

    environment {
        JAVA_HOME = tool 'JDK17'
        MAVEN_HOME = tool 'Maven'
        PATH = "${JAVA_HOME}/bin:${MAVEN_HOME}/bin:${env.PATH}"
        APP_NAME = 'jenkins-demo'
        ARTIFACT_PATH = 'target'
        STAGING_DEPLOY_DIR = '/opt/apps/${APP_NAME}/staging'
        PROD_DEPLOY_DIR = '/opt/apps/${APP_NAME}/prod'
        STAGING_URL = 'http://127.0.0.1:8081'
        PROD_URL = 'http://127.0.0.1:8082'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipStagesAfterUnstable()
    }

    stages {
        stage('Checkout') {
            steps {
                echo '拉取代码'
                checkout scm
            }
        }

        stage('Compile & Unit Test') {
            steps {
                echo '编译并执行单元测试'
                sh ' mvn -B  -U clean test'
            }
        }

        stage('Integration Test') {
            steps {
                echo '执行集成测试'
                sh ' mvn -B  verify -DskipTests=false'
            }
        }

        stage('Package Artifact') {
            steps {
                echo '打包应用'
                sh ' mvn -B  package -DskipTests'
                archiveArtifacts artifacts: "${ARTIFACT_PATH}/*.jar", fingerprint: true
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo '部署到测试环境'
                sh '''
                    set -eux
                    mkdir -p "$STAGING_DEPLOY_DIR"
                    cp -f "$ARTIFACT_PATH"/*.jar "$STAGING_DEPLOY_DIR"/
                    echo "Deploy artifact to staging: $STAGING_DEPLOY_DIR"
                '''
            }
        }

        stage('Smoke Test on Staging') {
            steps {
                echo '验证测试环境健康状态'
                sh '''
                    set -eux
                    curl -fsSL "$STAGING_URL/actuator/health" || curl -fsSL "$STAGING_URL/health" || curl -fsSL "$STAGING_URL"
                '''
            }
        }

        stage('Production Approval') {
            when {
                branch 'main'
            }
            steps {
                echo '等待生产环境发布审批'
                input message: '是否部署到生产环境？', ok: '确认部署'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo '部署到生产环境'
                sh '''
                    set -eux
                    mkdir -p "$PROD_DEPLOY_DIR"
                    cp -f "$ARTIFACT_PATH"/*.jar "$PROD_DEPLOY_DIR"/
                    echo "Deploy artifact to production: $PROD_DEPLOY_DIR"
                '''
            }
        }

        stage('Production Verification') {
            when {
                branch 'main'
            }
            steps {
                echo '验证生产环境是否正常运行'
                sh '''
                    set -eux
                    curl -fsSL "$PROD_URL/actuator/health" || curl -fsSL "$PROD_URL/health" || curl -fsSL "$PROD_URL"
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
            
        }
        success {
            echo '构建和部署成功'
        }
        failure {
            echo '构建或部署失败，请检查日志'
        }
        unstable {
            echo '构建不稳定，需人工确认'
        }
    }
}
