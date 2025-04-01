pipeline {
    agent any
    
    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        TERRAFORM_DIR = 'terraform'
        ANSIBLE_DIR = 'ansible'
        AWS_REGION = 'us-east-1'
        WORKSPACE = 'C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Promotion-Website'
        NO_PROXY = '*.docker.io,registry-1.docker.io'
        WSL_SSH_KEY = '/root/.ssh/Promotion-Website.pem'
        TERRAFORM_PARALLELISM = '10'
        GIT_PATH = 'C:\\Program Files\\Git\\bin\\git.exe'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/udaraDev/Promotion-Website.git'
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred',
                         usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        def gitCommitHash = bat(
                            script: "\"${env.GIT_PATH}\" rev-parse --short HEAD",
                            returnStdout: true
                        ).trim().readLines().last()
                        
                        // Build backend image
                        bat """
                            docker build -t ${DOCKER_HUB_USERNAME}/dairy-backend:latest -t ${DOCKER_HUB_USERNAME}/dairy-backend:${gitCommitHash} ./backend
                        """
                        
                        // Build frontend image
                        bat """
                            docker build -t ${DOCKER_HUB_USERNAME}/dairy-frontend:latest -t ${DOCKER_HUB_USERNAME}/dairy-frontend:${gitCommitHash} ./frontend
                        """
                    }
                }
            }
        }
        
        stage('Push Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred',
                         usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        def gitCommitHash = bat(
                            script: "\"${env.GIT_PATH}\" rev-parse --short HEAD",
                            returnStdout: true
                        ).trim().readLines().last()
                        
                        // Login to Docker Hub
                        bat "echo %DOCKER_HUB_PASSWORD% | docker login -u %DOCKER_HUB_USERNAME% --password-stdin"
                        
                        // Push backend images
                        retry(3) {
                            bat """
                                docker push ${DOCKER_HUB_USERNAME}/dairy-backend:${gitCommitHash}
                                docker push ${DOCKER_HUB_USERNAME}/dairy-backend:latest
                            """
                        }
                        
                        // Push frontend images
                        retry(3) {
                            bat """
                                docker push ${DOCKER_HUB_USERNAME}/dairy-frontend:${gitCommitHash}
                                docker push ${DOCKER_HUB_USERNAME}/dairy-frontend:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Terraform Apply') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        script {
                            def maxRetries = 3
                            def retryCount = 0
                            def success = false
                            
                            // Set Terraform environment variables
                            env.TF_CLI_TIMEOUT = "120"
                            env.TF_HTTP_TIMEOUT = "120"
                            
                            // Clear proxy settings
                            env.HTTP_PROXY = ""
                            env.HTTPS_PROXY = ""
                            env.NO_PROXY = "registry.terraform.io,releases.hashicorp.com"
                            
                            while (!success && retryCount < maxRetries) {
                                try {
                                    // First try to clean up any existing resources
                                    bat """
                                        terraform init -input=false
                                        
                                        :: Try to destroy any existing infrastructure
                                        terraform destroy -auto-approve || exit 0
                                        
                                        :: Clean up terraform state
                                        if exist ".terraform" rmdir /s /q .terraform
                                        if exist ".terraform.lock.hcl" del /f .terraform.lock.hcl
                                        if exist "terraform.tfstate" del /f terraform.tfstate
                                        if exist "terraform.tfstate.backup" del /f terraform.tfstate.backup
                                        
                                        :: Reinitialize and apply
                                        terraform init -input=false
                                        terraform apply -auto-approve -parallelism=${TERRAFORM_PARALLELISM}
                                    """
                                    
                                    // Get the EC2 IP
                                    env.EC2_IP = bat(
                                        script: 'terraform output -raw public_ip',
                                        returnStdout: true
                                    ).trim()
                                    
                                    success = true
                                } catch (Exception e) {
                                    retryCount++
                                    echo "Attempt ${retryCount} failed: ${e.message}"
                                    
                                    if (e.message.contains("VpcLimitExceeded")) {
                                        error """
                                            VPC limit exceeded. Please:
                                            1. Log into AWS Console
                                            2. Go to VPC Dashboard
                                            3. Delete unused VPCs or request limit increase
                                            Current error: ${e.message}
                                        """
                                    } else if (e.message.contains("connection attempt failed") || 
                                             e.message.contains("deadline exceeded")) {
                                        echo "Network connectivity issue detected. Waiting longer before retry..."
                                        sleep(60)
                                    } else if (retryCount >= maxRetries) {
                                        error "Failed to apply Terraform configuration after ${maxRetries} attempts: ${e.message}"
                                    } else {
                                        echo "Waiting 30 seconds before retrying..."
                                        sleep(30)
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Ansible Deployment') {
            steps {
                dir(ANSIBLE_DIR) {
                script {
                        if (!env.EC2_IP?.trim()) {
                            error "EC2 IP address not set. Cannot proceed with deployment."
                        }
                        
                        // Create .ssh directory in Jenkins workspace using proper Windows commands
                        bat """
                            if not exist "%WORKSPACE%\\.ssh" (
                                mkdir "%WORKSPACE%\\.ssh"
                                icacls "%WORKSPACE%\\.ssh" /inheritance:r
                                icacls "%WORKSPACE%\\.ssh" /grant "SYSTEM:(OI)(CI)F"
                                icacls "%WORKSPACE%\\.ssh" /grant "Administrators:(OI)(CI)F"
                                icacls "%WORKSPACE%\\.ssh" /grant "JENKINS:(OI)(CI)F"
                            )
                        """
                        
                        // Copy SSH key to workspace with proper permissions
                        withCredentials([file(credentialsId: 'promotion-website-pem', variable: 'SSH_KEY')]) {
                            bat """
                                copy /Y "%SSH_KEY%" "%WORKSPACE%\\.ssh\\Promotion-Website.pem"
                                icacls "%WORKSPACE%\\.ssh\\Promotion-Website.pem" /inheritance:r
                                icacls "%WORKSPACE%\\.ssh\\Promotion-Website.pem" /grant "SYSTEM:R"
                                icacls "%WORKSPACE%\\.ssh\\Promotion-Website.pem" /grant "Administrators:R"
                                icacls "%WORKSPACE%\\.ssh\\Promotion-Website.pem" /grant "JENKINS:R"
                            """
                        }

                        withCredentials([usernamePassword(credentialsId: 'dockerhub-cred',
                                     usernameVariable: 'DOCKER_HUB_USERNAME',
                                     passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                            def gitCommitHash = bat(
                                script: "\"${env.GIT_PATH}\" rev-parse --short HEAD",
                                returnStdout: true
                            ).trim().readLines().last()
                            
                            // Create inventory file
                            writeFile file: 'inventory.ini', text: """[ec2]
${env.EC2_IP} ansible_user=ubuntu ansible_ssh_private_key_file=${WORKSPACE}\\.ssh\\Promotion-Website.pem ansible_connection=ssh

[ec2:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o ConnectTimeout=60'
"""

                            // Run Ansible playbook
                            bat """
                                set ANSIBLE_HOST_KEY_CHECKING=False
                                set ANSIBLE_CONFIG=%WORKSPACE%\\ansible\\ansible.cfg
                                ansible-playbook -i inventory.ini deploy.yml ^
                                    -e "docker_hub_username=%DOCKER_HUB_USERNAME%" ^
                                    -e "git_commit_hash=${gitCommitHash}" ^
                                    -vvv
                            """
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
            bat 'docker logout'
                cleanWs(
                    cleanWhenSuccess: true,
                    cleanWhenFailure: true,
                    cleanWhenAborted: true
                )
            }
        }
        success {
            echo 'Deployment completed successfully!'
            echo "Frontend URL: http://${env.EC2_IP}:5173"
            echo "Backend URL: http://${env.EC2_IP}:4000"
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}