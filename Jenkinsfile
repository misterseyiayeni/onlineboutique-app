def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-west-2'
    }

    stages {
        // Checkout To The Microservice Branch
        stage('Checkout To Microservice Branch') {
            steps {
                git branch: 'app-database', url: 'https://github.com/Dappyplay4u/multi-microservices-application-projects.git'
            }
        }

        // Configure AWS CLI before deployment
        stage('Configure AWS CLI') {
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
                                echo "✅ Verifying AWS credentials..."
                                aws sts get-caller-identity
                            '''
                        }
                    }
                }
            }
        }

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