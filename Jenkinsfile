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
        WSL_SSH_KEY = '/home/gayanshaminda2001/ansible/E-commerece-key-pair.pem'
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
                        
                        // Configure Docker to bypass proxy for Docker Hub
                        bat '''
                            echo {"proxies":{"default":{"httpProxy":"","httpsProxy":"","noProxy":"*.docker.io,registry-1.docker.io"}}} > %USERPROFILE%\\.docker\\config.json
                        '''
                        
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

        
        stage('Terraform Initialize') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        // Use Windows terraform consistently
                        bat "terraform init -input=false -upgrade"
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        script {
                            // Check if output file exists using Windows commands
                            def outputExists = bat(
                                script: 'terraform output -json >nul 2>&1 && echo exists || echo notexists', 
                                returnStdout: true
                            ).trim().contains("exists")
                            
                            if (outputExists) {
                                try {
                                    env.EXISTING_EC2_IP = bat(
                                        script: 'terraform output -raw public_ip 2>nul || echo ""', 
                                        returnStdout: true
                                    ).trim().readLines().last()
                                    echo "Found existing infrastructure with IP: ${env.EXISTING_EC2_IP}"
                                } catch (Exception e) {
                                    env.EXISTING_EC2_IP = ""
                                    echo "No existing infrastructure detected"
                                }
                            }
                            
                            // Run terraform plan to detect changes (using Windows Terraform)
                            def planExitCode = bat(
                                script: "terraform plan -detailed-exitcode -var=\"region=${AWS_REGION}\" -out=tfplan", 
                                returnStatus: true
                            )
                            env.TERRAFORM_CHANGES = planExitCode == 2 ? 'true' : 'false'
                            
                            if (env.TERRAFORM_CHANGES == 'true') {
                                echo "Infrastructure changes detected - will apply changes"
                            } else {
                                echo "No infrastructure changes detected - will skip apply stage"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Terraform Apply') {
            when {
                expression { return env.TERRAFORM_CHANGES == 'true' }
            }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        script {
                            // Apply using Windows Terraform to match plan version
                            bat "terraform apply -parallelism=${TERRAFORM_PARALLELISM} -input=false tfplan"
                            
                            // Get the EC2 IP using Windows Terraform
                            env.EC2_IP = bat(
                                script: 'terraform output -raw public_ip',
                                returnStdout: true
                            ).trim().readLines().last()
                            
                            // Add a small wait for EC2 instance to initialize
                            echo "Waiting 30 seconds for EC2 instance to initialize..."
                            sleep(30)
                        }
                    }
                }
            }
        }

        stage('Get Existing Infrastructure') {
            when {
                expression { return env.TERRAFORM_CHANGES == 'false' && env.EXISTING_EC2_IP?.trim() }
            }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        script {
                            env.EC2_IP = env.EXISTING_EC2_IP
                            echo "Using existing infrastructure with IP: ${env.EC2_IP}"
                        }
                    }
                }
            }
        }

        stage('Prepare SSH Key') {
            steps {
                script {
                    // Create directories if they don't exist
                    bat '''
                        if not exist ansible mkdir ansible
                        wsl mkdir -p /home/myuser/.ssh
                    '''
                    
                    // Set proper permissions on existing key
                    bat '''
                        wsl chmod 600 /home/myuser/.ssh/Promotion-Website.pem
                        wsl ls -la /home/myuser/.ssh/Promotion-Website.pem
                    '''
                    
                    // Verify SSH connection
                    bat '''
                        wsl ssh -o StrictHostKeyChecking=no -i /home/myuser/.ssh/Promotion-Website.pem ubuntu@13.218.143.183 "echo SSH connection successful"
                    '''
                }
            }
        }
        
       stage('Ansible Deployment') {
            steps {
                dir(ANSIBLE_DIR) {
                    withCredentials([
                        usernamePassword(credentialsId: 'dockerhub-cred',
                                     usernameVariable: 'DOCKER_HUB_USERNAME',
                                     passwordVariable: 'DOCKER_HUB_PASSWORD')
                    ]) {
                        script {
                            // Use the full path to Git executable in WSL
                            def gitCommitHash = bat(
                                script: "wsl git rev-parse --short HEAD",
                                returnStdout: true
                            ).trim().readLines().last()
                            
                            // Create inventory file
                            writeFile file: 'temp_inventory.ini', text: """[ec2]
${EC2_IP} ansible_user=ubuntu ansible_ssh_private_key_file=${WSL_SSH_KEY}

[ec2:vars]
ansible_ssh_private_key_file=/root/.ssh/Promotion-Website.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
"""
                            // Run Ansible playbook with corrected command format
                            def result = bat(
                                script: "wsl ansible-playbook -i temp_inventory.ini deploy.yml -u ubuntu --private-key ${WSL_SSH_KEY} -e \"DOCKER_HUB_USERNAME=${DOCKER_HUB_USERNAME} GIT_COMMIT_HASH=${gitCommitHash}\" -vvv",
                                returnStatus: true
                            )
                            
                            if (result != 0) {
                                error "Ansible deployment failed with exit code ${result}"
                            }
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
                    cleanWhenAborted: true,
                    patterns: [[pattern: '**/.git/**', type: 'EXCLUDE']]
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