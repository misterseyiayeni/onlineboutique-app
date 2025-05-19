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
        gradle 'Gradle'
    }

    stages {
        // // Run Gradle SonarQube Scan (if applicable)
        // stage('SonarQube Inspection') {
        //     steps {
        //         sh 'gradle sonarqube'
        //     }
        // }

        // // SonarQube SAST Code Analysis
        // stage("SonarQube SAST Analysis") {
        //     steps {
        //         withSonarQubeEnv('Sonar-Server') {
        //             sh ''' 
        //                 $SCANNER_HOME/bin/sonar-scanner \
        //                 -Dsonar.projectName=app-ad-serverice \
        //                 -Dsonar.projectKey=app-ad-serverice
        //             '''
        //         }
        //     }
        // }

        // Providing Snyk Access
        stage('Authenticate & Authorize Snyk') {
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token', variable: 'SNYK_TOKEN')]) {
                    sh "${SNYK_HOME}/snyk-linux auth $SNYK_TOKEN"
                }
            }
        }

        // OPA Dockerfile Security Scan
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
                        sh "docker build -t misterseyiayeni/adservice:latest ."
                    }
                }
            }
        }

        // Snyk SCA Test
        stage('Snyk SCA Test | Dependencies') {
            steps {
                sh "${SNYK_HOME}/snyk-linux test --docker misterseyiayeni/adservice:latest || true"
            }
        }

        // Push Image to DockerHub
        stage('Push Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker push misterseyiayeni/adservice:latest"
                    }
                }
            }
        }

        // Configure AWS CLI before Kubernetes operations
        stage('Configure AWS CLI') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            aws configure set region $AWS_DEFAULT_REGION
                            aws sts get-caller-identity
                            aws eks update-kubeconfig --name online-shop-eks-cluster --region us-west-2
                        """
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
                        sh 'aws eks update-kubeconfig --name online-shop-eks-cluster --region us-west-2'
                        sh 'kubectl apply -f deploy-envs/test-env/deployment.yaml'
                        sh 'kubectl apply -f deploy-envs/test-env/service.yaml'
                    }
                }
            }
        }

        // Manual Approval for Production
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
            slackSend channel: '#sa-devsecops-cicd-alerts',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info at: ${env.BUILD_URL}"
        }
    }
}
