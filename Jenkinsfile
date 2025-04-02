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
        WSL_SSH_KEY = '/home/myuser/.ssh/key.pem' // Adjust this path to your SSH key location in WSL
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/udaraDev/Promotion-Website.git'
            }
        }
        
        stage('Test') {
            steps {
                dir('frontend') {
                    script {
                        // Install dependencies
                        bat 'npm install'
                        // Run tests
                        bat 'npm test'
                    }
                }
                dir('backend') {
                    script {
                        // Install dependencies
                        bat 'npm install'
                        // Run tests
                        bat 'npm test'
                    }
                }
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
                        bat "wsl terraform init -input=false -upgrade"
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            steps {
                dir(TERRAFORM_DIR) {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-credentials',
                         accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                         secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        script {
                            // Check if output file exists using Windows-compatible commands
                            def outputExists = bat(script: 'wsl terraform output -json > nul 2>&1 && echo exists || echo notexists', returnStdout: true).trim().contains("exists")
                            
                            if (outputExists) {
                                // Try to get existing IP to check if infrastructure exists
                                try {
                                    env.EXISTING_EC2_IP = bat(script: 'wsl terraform output -raw public_ip 2> nul || echo ""', returnStdout: true).trim().readLines().last()
                                    echo "Found existing infrastructure with IP: ${env.EXISTING_EC2_IP}"
                                } catch (Exception e) {
                                    env.EXISTING_EC2_IP = ""
                                    echo "No existing infrastructure detected"
                                }
                            }
                            
                            // Run terraform plan with detailed exit code to automatically detect changes
                            def planExitCode = bat(script: "wsl terraform plan -detailed-exitcode -var=\"region=${AWS_REGION}\" -out=tfplan", returnStatus: true)
                            // Exit code 0 = No changes, 1 = Error, 2 = Changes present
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
                        // Apply with parallelism for faster resource creation
                        bat "wsl terraform apply -parallelism=${TERRAFORM_PARALLELISM} -input=false tfplan"
                        
                        // Capture the EC2 IP from Terraform output
                        script {
                            env.EC2_IP = bat(script: 'wsl terraform output -raw public_ip', returnStdout: true).trim().readLines().last()
                            // Add a small wait for EC2 instance to initialize (reduces SSH connection issues)
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

         stage(' Ansible Deployment') {
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
                                script: "wsl \"${env.GIT_PATH}\" rev-parse --short HEAD",
                                returnStdout: true
                            ).trim().readLines().last()
                            
                            // Create inventory file
                            writeFile file: 'temp_inventory.ini', text: """[ec2]
${EC2_IP} ansible_user=ubuntu ansible_ssh_private_key_file=${WSL_SSH_KEY}

[ec2:vars]
ansible_python_interpreter=/usr/bin/python3
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
                try {
                    // First try to logout from Docker
                    bat 'docker logout'
                    
                    // Try to clean up workspace with more specific patterns
                    cleanWs(
                        deleteDirs: true,
                        patterns: [
                            [pattern: '**/.git/**', type: 'EXCLUDE'],
                            [pattern: 'terraform/terraform.tfstate', type: 'EXCLUDE'],
                            [pattern: 'terraform/terraform.tfstate.backup', type: 'EXCLUDE'],
                            [pattern: 'ansible/inventory.ini', type: 'EXCLUDE'],
                            [pattern: 'ansible/deploy.yml', type: 'EXCLUDE']
                        ],
                        notMatch: true
                    )
                    
                    // If the above fails, try to clean up specific directories
                    try {
                        bat '''
                            if exist frontend\\node_modules rmdir /s /q frontend\\node_modules
                            if exist backend\\node_modules rmdir /s /q backend\\node_modules
                            if exist .terraform rmdir /s /q .terraform
                            if exist terraform\\.terraform rmdir /s /q terraform\\.terraform
                        '''
                    } catch (Exception e) {
                        echo "Warning: Failed to clean up some directories: ${e.getMessage()}"
                    }
                } catch (Exception e) {
                    echo "Warning: Workspace cleanup encountered issues: ${e.getMessage()}"
                }
            }
        }
        success {
            echo "Deployment completed successfully! Frontend: http://${env.EC2_IP}:5173, Backend: http://${env.EC2_IP}:4000"
        }
        failure {
            echo "Deployment failed!"
            // Add more detailed error reporting
            script {
                if (currentBuild.result == 'FAILURE') {
                    echo "Pipeline failed at stage: ${currentBuild.currentStage}"
                    echo "Build number: ${currentBuild.number}"
                    echo "Build URL: ${currentBuild.absoluteUrl}"
                }
            }
        }
    }
}