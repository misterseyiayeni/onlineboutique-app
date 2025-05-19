def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any

    environment {
        SCANNER_HOME=tool 'SonarScanner'
        SNYK_HOME   = tool name: 'Snyk'
        AWS_DEFAULT_REGION = 'us-west-2'
    }

    tools {
        snyk 'Snyk'
    }

    stages {
        stage("SonarQube SAST Analysis") {
            steps {
                withSonarQubeEnv('Sonar-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=app-frontend-service \
                          -Dsonar.projectKey=app-frontend-service'''
                }
            }
        }

        stage('Authenticate & Authorize Snyk') {
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token', variable: 'SNYK_TOKEN')]) {
                    sh "${SNYK_HOME}/snyk-linux auth $SNYK_TOKEN"
                }
            }
        }

        stage('OPA Dockerfile Vulnerability Scan') {
            steps {
                sh "docker run --rm -v ${WORKSPACE}:/project openpolicyagent/conftest test --policy docker-opa-security.rego Dockerfile || true"
            }
        }

        stage('Build & Tag Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker build -t misterseyiayeni/frontendservice:latest ."
                    }
                }
            }
        }

        stage('Snyk SCA Test | Dependencies') {
            steps {
                sh "${SNYK_HOME}/snyk-linux test --docker misterseyiayeni/frontendservice:latest || true" 
            }
        }

        stage('Push Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker push misterseyiayeni/frontendservice:latest"
                    }
                }
            }
        }

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

        stage('Deploy Microservice to Stage/Test Env') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'Kubernetes-Credential') {
                        sh 'aws eks update-kubeconfig --name online-shop-eks-cluster --region us-west-2'
                        sh 'kubectl apply -f deploy-envs/test-env/deployment.yaml --validate=false'
                        sh 'kubectl apply -f deploy-envs/test-env/nodeport-service.yaml --validate=false'
                    }
                }
            }
        }

        stage('ZAP Dynamic Testing | DAST') {
            steps {
                sshagent(['OWASP-Zap-Credential']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@35.86.104.93 "docker run -t zaproxy/zap-weekly zap-baseline.py -t http://54.190.237.169:30000/" || true'
                }
            }
        }

        stage('Approve Prod Deployment') {
            steps {
                input('Do you want to proceed?')
            }
        }

        stage('Deploy Microservice to Prod Env') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'Kubernetes-Credential') {
                        sh 'kubectl apply -f deploy-envs/prod-env/deployment.yaml'
                        sh 'kubectl apply -f deploy-envs/prod-env/loadbalancer-service.yaml'
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#sa-devsecops-cicd-alerts',
                      color: COLOR_MAP[currentBuild.currentResult],
                      message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info at: ${env.BUILD_URL}"
        }
    }
}
