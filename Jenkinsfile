pipeline {
    agent any

    environment {
        APP_REPO      = 'https://github.com/karishmagdly27/demo-pro1.git'
        APP_NAME      = 'demo-app'
        DEV_HOST      = '3.134.102.126'
        STAGING_HOST  = '3.134.88.210'
        PROD_HOST     = '18.191.129.250'
        SONAR_HOST    = 'http://13.59.224.139:9000' 
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'dev'
                    git branch: branch, url: "${APP_REPO}", credentialsId: 'git'
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { return ['dev','stg','master'].contains(env.BRANCH_NAME) } }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${APP_NAME}-${env.BRANCH_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST}
                    """
                }
            }
        }

        stage('Package App') {
            steps {
                echo "Packaging frontend application..."
                sh """
                    rm -f /tmp/${APP_NAME}.zip
                    zip -r /tmp/${APP_NAME}.zip ./*
                """
            }
        }

        stage('Upload to Nexus') {
            when { expression { return ['stg','master'].contains(env.BRANCH_NAME) } }
            steps {
                nexusArtifactUploader artifacts: [[
                        artifactId: "${APP_NAME}",
                        file: "/tmp/${APP_NAME}.zip",
                        type: 'zip'
                    ]],
                    credentialsId: 'nexus-creds',
                    groupId: 'com.demo',
                    nexusUrl: 'http://3.145.4.201:8081',
                    repository: 'demo-app-repo',
                    version: "${env.BRANCH_NAME}-${BUILD_NUMBER}"
            }
        }

        stage('Deploy') {
            when { expression { return ['dev','stg','master'].contains(env.BRANCH_NAME) } }
            steps {
                script {
                    def targets = [
                        "dev": DEV_HOST,
                        "stg": STAGING_HOST,
                        "master": PROD_HOST
                    ]
                    def host = targets[env.BRANCH_NAME]

                    if (env.BRANCH_NAME == "master") {
                        input message: "Deploy to Production?", ok: "Deploy"
                    }

                    deployToServer(host)
                }
            }
        }
    }

    post {
        always { echo "Pipeline completed." }
        success { echo "Pipeline succeeded for ${env.BRANCH_NAME}!" }
        failure { echo "Pipeline FAILED." }
    }
}

def deployToServer(String host) {
    sshagent(['app-key']) {
        sh """
        echo "🚀 Deploying ${APP_NAME} to ${host}"

        ssh -o StrictHostKeyChecking=no ubuntu@${host} '
            sudo rm -rf /var/www/html/* &&
            sudo unzip -o /tmp/${APP_NAME}.zip -d /var/www/html &&
            sudo chown -R www-data:www-data /var/www/html &&
            sudo chmod -R 755 /var/www/html &&
            sudo systemctl reload nginx
        '
        """
    }
}
