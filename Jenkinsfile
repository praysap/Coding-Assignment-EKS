pipeline {
    agent any
    
    environment {
        REPO_URL = 'https://github.com/vipulsaw/Container-Orchestration-2.git'
        REPO_NAME = 'Container-Orchestration-2'
        HELM_CHART_DIR = 'mern-app'
        NAMESPACE = 'mern'
        RELEASE_NAME = 'mern-app'
        EC2_INSTANCE_IP = '13.203.66.123'
        DOCKER_HUB_REPO = 'vipulsaw123'
    }
    
    stages {
        stage('Checkout Source Code') {
            steps {
                checkout([$class: 'GitSCM', 
                         branches: [[name: '*/main']], 
                         userRemoteConfigs: [[url: env.REPO_URL]]])
                echo "Repository cloned successfully"
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    docker.build("${env.DOCKER_HUB_REPO}/learner-frontend:latest", './learnerReportCS_frontend')
                    docker.build("${env.DOCKER_HUB_REPO}/learner-backend:latest", './learnerReportCS_backend')
                    echo "Docker images built successfully"
                }
            }
        }
        
        stage('Push Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials-Vipul', 
                        usernameVariable: 'DOCKER_HUB_USER', 
                        passwordVariable: 'DOCKER_HUB_PASSWORD'
                    )]) {
                        sh "echo \$DOCKER_HUB_PASSWORD | docker login -u \$DOCKER_HUB_USER --password-stdin"
                        sh "docker push ${env.DOCKER_HUB_REPO}/learner-frontend:latest"
                        sh "docker push ${env.DOCKER_HUB_REPO}/learner-backend:latest"
                        echo "Docker images pushed to Docker Hub successfully"
                    }
                }
            }
        }
        
        stage('Copy Repository to Target EC2') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'mkdir -p ~/${env.REPO_NAME}'
                        """
                        sh """
                            rsync -avz -e "ssh -o StrictHostKeyChecking=no" \
                            --exclude='.git' \
                            --delete \
                            ./ root@${env.EC2_INSTANCE_IP}:~/${env.REPO_NAME}/
                        """
                        echo "Files copied to EC2 instance successfully"
                    }
                }
            }
        }
        
        stage('Install/Upgrade Helm Release') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'cd ~/${env.REPO_NAME}/${env.HELM_CHART_DIR} && \
                            helm upgrade --install ${env.RELEASE_NAME} . \
                            --namespace ${env.NAMESPACE} \
                            --create-namespace'
                        """
                        echo "Helm release upgraded successfully"
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'helm list -n ${env.NAMESPACE} && \
                            kubectl get pods -n ${env.NAMESPACE}'
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
    }
}
