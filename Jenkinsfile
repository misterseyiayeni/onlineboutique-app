def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarScanner'
        SNYK_HOME    = tool name: 'Snyk'
        AWS_DEFAULT_REGION = 'us-west-2'
    }

    tools {
        snyk 'Snyk'
    }

    stages {
        // SonarQube SAST Code Analysis
        stage("SonarQube SAST Analysis") {
            steps {
                withSonarQubeEnv('Sonar-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=app-checkout-service \
                        -Dsonar.projectKey=app-checkout-service'''
                }
            }
        }

        // Authenticate & Authorize Snyk
        stage('Authenticate & Authorize Snyk') {
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token', variable: 'SNYK_TOKEN')]) {
                    sh "${SNYK_HOME}/snyk-linux auth $SNYK_TOKEN"
                }
            }
        }

        // OPA Dockerfile Scan
        stage('OPA Dockerfile Vulnerability Scan') {
            steps {
                sh "docker run --rm -v ${WORKSPACE}:/project openpolicyagent/conftest test --policy docker-opa-security.rego Dockerfile || true"
            }
        }

        // Build & Tag Docker Image
        stage('Build & Tag Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker build -t misterseyiayeni/checkoutservice:latest ."
                    }
                }
            }
        }

        // Snyk SCA Test
        stage('Snyk SCA Test | Dependencies') {
            steps {
                sh "${SNYK_HOME}/snyk-linux test --docker misterseyiayeni/checkoutservice:latest || true"
            }
        }

        // Push Image to DockerHub
        stage('Push Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker push misterseyiayeni/checkoutservice:latest"
                    }
                }
            }
        }

        // Configure AWS CLI
        stage('Configure AWS CLI') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            aws configure set region $AWS_DEFAULT_REGION
                        '''
                    }
                }
            }
        }

        // Deploy to Staging/Test Environment
        stage('Deploy Microservice To The Stage/Test Env') {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: '',
                        contextName: '',
                        credentialsId: 'Kubernetes-Credential',
                        namespace: '',
                        restrictKubeConfigAccess: false,
                        serverUrl: ''
                    ) {
                        sh 'kubectl apply -f deploy-envs/test-env/deployment.yaml'
                        sh 'kubectl apply -f deploy-envs/test-env/service.yaml'
                    }
                }
            }
        }

        // Production Deployment Approval
        stage('Approve Prod Deployment') {
            steps {
                input('Do you want to proceed?')
            }
        }

        // Deploy to Production Environment
        stage('Deploy Microservice To The Prod Env') {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: '',
                        contextName: '',
                        credentialsId: 'Kubernetes-Credential',
                        namespace: '',
                        restrictKubeConfigAccess: false,
                        serverUrl: ''
                    ) {
                        sh 'kubectl apply -f deploy-envs/prod-env/deployment.yaml'
                        sh 'kubectl apply -f deploy-envs/prod-env/service.yaml'
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#sa-devsecops-cicd-alerts', //update and provide your channel name
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \nBuild Timestamp: ${env.BUILD_TIMESTAMP} \nProject Workspace: ${env.WORKSPACE} \nMore info at: ${env.BUILD_URL}"
        }
    }
}