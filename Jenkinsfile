pipeline {
    agent any

    environment {
        IMAGE        = 'ushakannaji/secureship'
        TAG          = "${env.BUILD_NUMBER}"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: 'https://github.com/usharanikannaji/secureship-devsecops-eks.git'
            }
        }
        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner \
                      -Dsonar.projectKey=secureship \
                      -Dsonar.projectName=secureship \
                      -Dsonar.sources=src \
                      -Dsonar.exclusions=node_modules/**,build/**'''
                }
            }
        }
        stage('Quality gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage('Trivy file scan') {
            steps {
                sh 'trivy fs --skip-version-check --no-progress --severity HIGH,CRITICAL --exit-code 0 . > trivy-fs.txt'
            }
        }
        stage('Docker build') {
            steps { sh 'docker build -t ${IMAGE}:${TAG} .' }
        }
        stage('Trivy image scan') {
            steps {
                sh 'trivy image --skip-version-check --no-progress --severity HIGH,CRITICAL --exit-code 0 ${IMAGE}:${TAG} > trivy-image.txt'
            }
        }
        stage('Push image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                          docker push ${IMAGE}:${TAG}'''
                }
            }
        }
        stage('Run container') {
            steps {
                sh '''docker rm -f secureship || true
                      docker run -d --name secureship -p 3000:80 ${IMAGE}:${TAG}'''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true
        }
    }
}
