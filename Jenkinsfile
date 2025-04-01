pipeline {
    agent any
    
    parameters {
        booleanParam(defaultValue: false, description: 'Create new infrastructure', name: 'CREATE_NEW_INFRASTRUCTURE')
        string(defaultValue: '', description: 'EC2 IP address (leave empty to use existing)', name: 'MANUAL_EC2_IP')
    }
    
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
        
        stage('Setup EC2 IP') {
            steps {
                script {
                    // If manual IP is provided, use it
                    if (params.MANUAL_EC2_IP?.trim()) {
                        env.EC2_IP = params.MANUAL_EC2_IP.trim()
                        echo "Using manually provided EC2 IP: ${env.EC2_IP}"
                    } else {
                        echo "No manual EC2 IP provided, will attempt to detect from infrastructure"
                    }
                }
            }
        }
        
        stage('Terraform Initialize') {
            when {
                expression { 
                    return !params.MANUAL_EC2_IP?.trim()
                }
            }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        // Use Windows terraform directly
                        bat "terraform init -input=false -upgrade"
                    }
                }
            }
        }
        
        stage('Get Infrastructure Info') {
            when {
                expression { !params.MANUAL_EC2_IP?.trim() }
            }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        script {
                            // Use a simple approach to get the IP - redirect to a file with no color
                            bat 'set NO_COLOR=true && terraform output -no-color -raw public_ip > ec2_ip.txt 2>terraformerror.txt || type terraformerror.txt'
                            
                            // Read the IP from the file, but check if it exists first
                            if (fileExists('ec2_ip.txt')) {
                                def ipContent = readFile('ec2_ip.txt').trim()
                                // Make sure it looks like an IP (basic validation)
                                if (ipContent =~ /^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$/) {
                                    env.EC2_IP = ipContent
                                    echo "Found existing infrastructure with valid IP: ${env.EC2_IP}"
                                    
                                    // Run a basic health check to see if the instance is responding
                                    def pingResult = bat(
                                        script: "ping -n 1 -w 5000 ${env.EC2_IP} > pingresult.txt 2>&1",
                                        returnStatus: true
                                    )
                                    
                                    if (pingResult == 0) {
                                        echo "Successfully pinged ${env.EC2_IP}"
                                    } else {
                                        echo "Warning: Could not ping ${env.EC2_IP}, instance might be down"
                                    }
                                } else {
                                    echo "EC2 IP found but it doesn't look valid: ${ipContent}"
                                }
                            } else {
                                echo "No existing infrastructure IP found"
                            }
                            
                            // If we still don't have an IP and we want to create infrastructure
                            if (!env.EC2_IP?.trim() && params.CREATE_NEW_INFRASTRUCTURE) {
                                bat "terraform plan -var=\"region=${AWS_REGION}\" -out=tfplan"
                                echo "Infrastructure plan created and will apply if requested"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Terraform Apply') {
            when {
                expression { 
                    return params.CREATE_NEW_INFRASTRUCTURE && !env.EC2_IP?.trim()
                }
            }
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        script {
                            // Apply with tfplan file
                            bat "terraform apply -parallelism=${TERRAFORM_PARALLELISM} -input=false tfplan"
                            
                            // Get the EC2 IP without color or formatting
                            bat 'set NO_COLOR=true && terraform output -no-color -raw public_ip > ec2_ip.txt'
                            env.EC2_IP = readFile('ec2_ip.txt').trim()
                            
                            echo "New EC2 instance created with IP: ${env.EC2_IP}"
                            
                            // Add a wait for EC2 instance to initialize
                            echo "Waiting 120 seconds for EC2 instance to initialize..."
                            sleep(120)
                        }
                    }
                }
            }
        }
        
        stage('Verify EC2 IP') {
            steps {
                script {
                    // Verify EC2_IP is set
                    if (!env.EC2_IP?.trim()) {
                        error "EC2 IP address not set. Please provide a manual EC2 IP or ensure infrastructure exists."
                    }
                    
                    echo "Using EC2 IP address: ${env.EC2_IP}"
                    
                    // Check if EC2_IP is a valid IP address
                    if (!(env.EC2_IP =~ /^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$/)) {
                        error "EC2 IP address (${env.EC2_IP}) is not a valid IP address."
                    }
                }
            }
        }
        
        stage('Ansible Deployment') {
            steps {
                dir(ANSIBLE_DIR) {
                    script {
                        echo "Starting deployment to EC2 instance at IP: ${env.EC2_IP}"
                        
                        // Create directories if they don't exist
                        bat '''
                            if not exist ansible mkdir ansible
                            wsl mkdir -p /home/myuser/.ssh
                            wsl mkdir -p /root/.ssh
                        '''
                        
                        // Copy PEM file to WSL locations if using credential
                        withCredentials([file(credentialsId: 'promotion-website-pem', variable: 'PEM_FILE')]) {
                            bat '''
                                copy %PEM_FILE% .\\Promotion-Website.pem
                                wsl cp /mnt/c/ProgramData/Jenkins/.jenkins/workspace/promotion-website-diary/ansible/Promotion-Website.pem /home/myuser/.ssh/
                                wsl cp /mnt/c/ProgramData/Jenkins/.jenkins/workspace/promotion-website-diary/ansible/Promotion-Website.pem /root/.ssh/
                            '''
                        }
                        
                        // Set proper permissions on the key files
                        bat '''
                            wsl chmod 600 /home/myuser/.ssh/Promotion-Website.pem
                            wsl chmod 600 /root/.ssh/Promotion-Website.pem
                        '''
                        
                        // Store the IP in a variable to ensure it's clean
                        def cleanIp = env.EC2_IP.trim()
                        
                        echo "Testing SSH connection to ${cleanIp}..."
                        def sshReady = false
                        def maxRetries = 10
                        
                        for (int i = 0; i < maxRetries && !sshReady; i++) {
                            def sshCmd = "wsl ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 -i /home/myuser/.ssh/Promotion-Website.pem ubuntu@${cleanIp} -C 'echo SSH_CONNECTION_SUCCESSFUL' || echo CONNECTION_FAILED"
                            
                            def sshResult = bat(
                                script: sshCmd,
                                returnStdout: true
                            ).trim()
                            
                            if (sshResult.contains("SSH_CONNECTION_SUCCESSFUL")) {
                                echo "SSH connection successful on attempt ${i+1}"
                                sshReady = true
                            } else {
                                echo "SSH connection attempt ${i+1}/${maxRetries} failed, waiting 30 seconds..."
                                sleep(30)
                            }
                        }

                        withCredentials([
                            usernamePassword(credentialsId: 'dockerhub-cred',
                                         usernameVariable: 'DOCKER_HUB_USERNAME',
                                         passwordVariable: 'DOCKER_HUB_PASSWORD')
                        ]) {
                            def gitCommitHash = bat(script: 'wsl git rev-parse --short HEAD', returnStdout: true).trim().readLines().last()
                            
                            // Create inventory file with improved settings
                            writeFile file: 'temp_inventory.ini', text: """[ec2]
${cleanIp}

[ec2:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/myuser/.ssh/Promotion-Website.pem
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o ConnectTimeout=180 -o ServerAliveInterval=30 -o ServerAliveCountMax=10'
ansible_ssh_retries=10
ansible_connection_timeout=300
ansible_timeout=300
"""
                            // Use a retry loop for the Ansible playbook
                            def maxAttempts = 3
                            def playbookSuccess = false
                            
                            for (int attempt = 1; attempt <= maxAttempts && !playbookSuccess; attempt++) {
                                echo "Ansible playbook execution attempt ${attempt}/${maxAttempts}"
                                
                                def result = bat(
                                    script: "wsl ANSIBLE_HOST_KEY_CHECKING=False ANSIBLE_TIMEOUT=180 ansible-playbook -i temp_inventory.ini deploy.yml -e \"DOCKER_HUB_USERNAME=tharuka2001 GIT_COMMIT_HASH=${gitCommitHash}\" -v",
                                    returnStatus: true
                                )
                                
                                if (result == 0) {
                                    echo "Ansible playbook executed successfully on attempt ${attempt}"
                                    playbookSuccess = true
                                } else {
                                    echo "Ansible playbook failed on attempt ${attempt} with exit code ${result}"
                                    if (attempt < maxAttempts) {
                                        echo "Waiting 60 seconds before retry..."
                                        sleep(60)
                                    }
                                }
                            }
                            
                            if (!playbookSuccess) {
                                error "Ansible deployment failed after ${maxAttempts} attempts"
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
            script {
                if (env.EC2_IP) {
                    echo 'Deployment completed successfully!'
                    echo "Frontend URL: http://${env.EC2_IP}:5173"
                    echo "Backend URL: http://${env.EC2_IP}:4000"
                } else {
                    echo 'Deployment completed but EC2 IP is not available.'
                }
            }
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}