pipeline {
    agent any
    
    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        TERRAFORM_DIR = 'terraform'
        ANSIBLE_DIR = 'ansible'
        AWS_REGION = 'us-east-1'
        WORKSPACE = 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Promotion-Website'
        NO_PROXY = '*.docker.io,registry-1.docker.io'
        TERRAFORM_PARALLELISM = '10'
        GIT_PATH = 'C:\\Program Files\\Git\\bin\\git.exe'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/udaraDev/Promotion-Website.git'
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        def gitCommitHash = bat(script: "\"${env.GIT_PATH}\" rev-parse --short HEAD", returnStdout: true).trim().readLines().last()
                        
                        bat """
                            docker build -t ${DOCKER_HUB_USERNAME}/dairy-backend:latest -t ${DOCKER_HUB_USERNAME}/dairy-backend:${gitCommitHash} ./backend
                            docker build -t ${DOCKER_HUB_USERNAME}/dairy-frontend:latest -t ${DOCKER_HUB_USERNAME}/dairy-frontend:${gitCommitHash} ./frontend
                        """
                    }
                }
            }
        }
        
        stage('Push Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        def gitCommitHash = bat(script: "\"${env.GIT_PATH}\" rev-parse --short HEAD", returnStdout: true).trim().readLines().last()
                        
                        bat "docker login -u %DOCKER_HUB_USERNAME% -p %DOCKER_HUB_PASSWORD%"
                        retry(3) {
                            bat """
                                docker push ${DOCKER_HUB_USERNAME}/dairy-backend:${gitCommitHash}
                                docker push ${DOCKER_HUB_USERNAME}/dairy-backend:latest
                                docker push ${DOCKER_HUB_USERNAME}/dairy-frontend:${gitCommitHash}
                                docker push ${DOCKER_HUB_USERNAME}/dairy-frontend:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Terraform Initialize') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        bat "terraform init -input=false -upgrade"
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        script {
                            def planExitCode = bat(script: "terraform plan -detailed-exitcode -var=\"region=${AWS_REGION}\" -out=tfplan", returnStatus: true)
                            
                            if (planExitCode == 2) {
                                env.TERRAFORM_CHANGES = 'true'
                            } else if (planExitCode == 0) {
                                env.TERRAFORM_CHANGES = 'false'
                            } else {
                                error "Terraform plan failed with exit code ${planExitCode}"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Terraform Apply') {
            when { expression { return env.TERRAFORM_CHANGES == 'true' } }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        script {
                            bat "terraform apply -parallelism=${TERRAFORM_PARALLELISM} -input=false tfplan"
                            env.EC2_IP = bat(script: 'terraform output -raw public_ip', returnStdout: true).trim().readLines().last()
                        }
                    }
                }
            }
        }
        
        stage('Get Existing Infrastructure') {
            when { expression { return env.TERRAFORM_CHANGES == 'false' && env.EXISTING_EC2_IP?.trim() } }
            steps {
                script {
                    env.EC2_IP = env.EXISTING_EC2_IP
                }
            }
        }
        
        stage('Ansible Deployment') {
            steps {
                dir(ANSIBLE_DIR) {
                    script {
                        if (!env.EC2_IP?.trim()) { error "EC2 IP not set. Cannot deploy." }
                        bat "wsl ansible-playbook -i inventory.ini deploy.yml -u ubuntu --private-key /home/myuser/.ssh/key.pem"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                bat 'docker logout'
                cleanWs()
            }
        }
        success {
            echo "Deployment completed successfully! Frontend: http://${env.EC2_IP}:5173, Backend: http://${env.EC2_IP}:4000"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
