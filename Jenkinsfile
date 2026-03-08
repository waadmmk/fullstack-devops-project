[3/8/2026 12:27 PM] Waad: pipeline {
    agent any

    environment {
        EMAIL_TO = "waad.majed777@gmail.com"
    }

    stages {

        stage('Build Backend') {
            steps {
                dir('demo') {
                    sh 'mvn clean package -DskipTests=true'
                }
            }
            post {
                success { notifyStage('Build Backend', 'SUCCESS') }
                failure { notifyStage('Build Backend', 'FAILURE') }
            }
        }

        stage('Test Backend') {
            environment {
                SPRING_PROFILES_ACTIVE = 'test-no-db'
            }
            steps {
                dir('demo') {
                    sh 'mvn test'
                }
            }
            post {
                success { notifyStage('Test Backend', 'SUCCESS') }
                failure { notifyStage('Test Backend', 'FAILURE') }
            }
        }

        stage('Install Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                }
            }
            post {
                success { notifyStage('Install Frontend', 'SUCCESS') }
                failure { notifyStage('Install Frontend', 'FAILURE') }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm run build'
                }
            }
            post {
                success { notifyStage('Build Frontend', 'SUCCESS') }
                failure { notifyStage('Build Frontend', 'FAILURE') }
            }
        }

        stage('Test Frontend') {
            steps {
                dir('frontend') {
                    sh 'xvfb-run -a npx ng test --watch=false --browsers=ChromeHeadless'
                }
            }
            post {
                success { notifyStage('Test Frontend', 'SUCCESS') }
                failure { notifyStage('Test Frontend', 'FAILURE') }
            }
        }

        stage('SonarQube') {
            parallel {

                stage('Sonar Backend') {
                    steps {
                        dir('demo') {
                            withSonarQubeEnv('sonarqube') {
                                sh '''
                                    mvn -DskipTests clean verify \
                                    org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                    -Dsonar.projectKey=fullstack-backend
                                '''
                            }
                        }
                    }
                    post {
                        success { notifyStage('Sonar Backend', 'SUCCESS') }
                        failure { notifyStage('Sonar Backend', 'FAILURE') }
                    }
                }

                stage('Sonar Frontend') {
                    steps {
                        dir('frontend') {
                            withSonarQubeEnv('sonarqube') {
                                script {
                                    def scannerHome = tool 'sonar-scanner'
                                    sh """
                                        ${scannerHome}/bin/sonar-scanner \
                                        -Dsonar.projectKey=fullstack-frontend \
                                        -Dsonar.sources=src
                                    """
                                }
                            }
                        }
                    }
                    post {
                        success { notifyStage('Sonar Frontend', 'SUCCESS') }
                        failure { notifyStage('Sonar Frontend', 'FAILURE') }
                    }
                }
            }
        }

        stage('Upload to Nexus') {
            parallel {

                stage('Upload Backend') {
                    steps {
                        dir('demo') {
                            nexusArtifactUploader artifacts: [
                                [
                                    artifactId: 'demo',
                                    classifier: '',
[3/8/2026 12:27 PM] Waad: file: 'target/demo-0.0.1-SNAPSHOT.jar',
                                    type: 'jar'
                                ]
                            ],
                            credentialsId: 'nexus',
                            groupId: 'com.example',
                            nexusUrl: '51.44.221.92:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'fullstack-backend',
                            version: "0.0.1-${BUILD_NUMBER}"
                        }
                    }
                    post {
                        success { notifyStage('Upload Backend', 'SUCCESS') }
                        failure { notifyStage('Upload Backend', 'FAILURE') }
                    }
                }

                stage('Upload Frontend') {
                    steps {
                        dir('frontend') {
                            sh "tar -czf frontend-${BUILD_NUMBER}.tgz dist"

                            nexusArtifactUploader artifacts: [
                                [
                                    artifactId: 'frontend',
                                    classifier: '',
                                    file: "frontend-${BUILD_NUMBER}.tgz",
                                    type: 'tgz'
                                ]
                            ],
                            credentialsId: 'nexus',
                            groupId: 'com.example',
                            nexusUrl: '15.188.28.246:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'fullstack-frontend',
                            version: "0.0.1-${BUILD_NUMBER}"
                        }
                    }
                    post {
                        success { notifyStage('Upload Frontend', 'SUCCESS') }
                        failure { notifyStage('Upload Frontend', 'FAILURE') }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DockerHub',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        ansible-playbook ansible/docker-playbook.yaml \
                        -e "workspace=$WORKSPACE tag=$BUILD_NUMBER"
                    '''
                }
            }
            post {
                success { notifyStage('Docker Build & Push', 'SUCCESS') }
                failure { notifyStage('Docker Build & Push', 'FAILURE') }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('ansible') {
                    sh 'ansible-playbook -i inventory.ini playbook-k8s.yaml'
                }
            }
            post {
                success { notifyStage('Deploy to Kubernetes', 'SUCCESS') }
                failure { notifyStage('Deploy to Kubernetes', 'FAILURE') }
            }
        }
    }

    post {
        success { notifyStage('PIPELINE', 'SUCCESS') }
        failure { notifyStage('PIPELINE', 'FAILURE') }
    }
}

def notifyStage(String stageName, String status) {
    emailext(
        to: env.EMAIL_TO,
        subject: "[Jenkins] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${stageName} | ${status}",
        mimeType: 'text/html',
        body: """
            <h3>Pipeline Notification</h3>
            <p><b>Job:</b> ${env.JOB_NAME}</p>
            <p><b>Build:</b> ${env.BUILD_NUMBER}</p>
            <p><b>Stage:</b> ${stageName}</p>
            <p><b>Status:</b> ${status}</p>
            <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
        """
    )
}
