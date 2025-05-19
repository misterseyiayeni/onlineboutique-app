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
    }
    tools {
        snyk 'Snyk'
    }
    stages {
        // SonarQube SAST Code Analysis
        stage("SonarQube SAST Analysis"){
            steps{
                withSonarQubeEnv('Sonar-Server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=app-product-catalog-service \
                    -Dsonar.projectKey=app-product-catalog-service '''
                }
            }
        }
        // Providing Snyk Access
        stage('Authenticate & Authorize Snyk') {
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token', variable: 'SNYK_TOKEN')]) {
                    sh "${SNYK_HOME}/snyk-linux auth $SNYK_TOKEN"
                }
            }
        }
        // Scan Service Dockerfile With Open Policy Agent (OPA)
        stage('OPA Dockerfile Vulnerability Scan') {
            steps {
                sh "docker run --rm -v ${WORKSPACE}:/project openpolicyagent/conftest test --policy docker-opa-security.rego Dockerfile || true"
            }
        }
        // Build and Tag Service Docker Image
        stage('Build & Tag Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker build -t misterseyiayeni/productcatalogservice:latest ."
                    }
                }
            }
        }
        // Execute SCA/Dependency Test on Service Docker Image
        stage('Snyk SCA Test | Dependencies') {
            steps {
                sh "${SNYK_HOME}/snyk-linux test --docker misterseyiayeni/productcatalogservice:latest || true" 
            }
        }
        // Push Service Image to DockerHub
        stage('Push Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker push misterseyiayeni/productcatalogservice:latest "
                    }
                }
            }
        }

        // // Configure AWS CLI before deployment
        // stage('Configure AWS CLI') {
        //     steps {
        //         script {
        //             withCredentials([usernamePassword(
        //                 credentialsId: 'aws-credentials',
        //                 usernameVariable: 'AWS_ACCESS_KEY_ID',
        //                 passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        //             )]) {
        //                 withEnv([
        //                     "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}",
        //                     "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}",
        //                     "AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}"
        //                 ]) {
        //                     sh '''
        //                         echo "✅ Verifying AWS credentials..."
        //                         aws sts get-caller-identity
        //                     '''
        //                 }
        //             }
        //         }
        //     }
        // }

        // Deploy to Staging
        stage('Deploy Microservice To The Stage/Test Env') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        withEnv([
                            "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}",
                            "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}",
                            "AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}"
                        ]) {
                            sh '''
                                echo "📥 Updating kubeconfig..."
                                aws eks update-kubeconfig --name online-shop-eks-cluster --region us-west-2

                                echo "📦 Deploying microservices to EKS..."
                                kubectl apply -f deploy-envs/test-env/deployment.yaml -v=7
                                kubectl apply -f deploy-envs/test-env/service.yaml -v=7
                            '''
                        }
                    }
                }
            }
        }

        // Manual Approval
        stage('Approve Prod Deployment') {
            steps {
                input('Do you want to proceed to production deployment?')
            }
        }

        // Deploy to Production
        stage('Deploy Microservice To The Prod Env') {
            steps {
                script {
                    sh '''
                        kubectl apply -f deploy-envs/prod-env/deployment.yaml
                        kubectl apply -f deploy-envs/prod-env/service.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Sending Slack Notification...'
            slackSend channel: '#sa-devsecops-cicd-alerts',
                color: COLOR_MAP.get(currentBuild.currentResult, 'warning'),
                message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' build #${env.BUILD_NUMBER} \n📅 ${new Date()} \n🔗 ${env.BUILD_URL}"
        }
    }
}