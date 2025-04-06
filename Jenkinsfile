pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/Shivanik799/bankapplication.git'
        GIT_BRANCH = 'main'
        DOCKER_IMAGE = 'shivanikanchukatla/finance'
        DOCKER_TAG = "${BUILD_NUMBER}"  // Use the Jenkins build number as the Docker tag
        AWS_REGION = 'us-east-1'
        ANSIBLE_PLAYBOOK = 'ansible-playbook.yml'
        CREDENTIALS_ID = 'github_pat' // Your credential ID here
        registryCredential = 'docker_pat'
        TEST_SERVER_IP = '10.0.101.92'  // Replace with your test server IP
        SSH_KEY_CREDENTIALS_ID = 'ssh_key'  
    }

    stages {
        stage('Checkout') {
            steps {
                // Use credentials for private repo
                git branch: "${GIT_BRANCH}", 
                    url: "${GIT_REPO}", 
                    credentialsId: "${CREDENTIALS_ID}" // Replace with your actual credential ID
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    echo "Docker image built: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        def dockerImage = docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}")
                        dockerImage.push()
                    }
                }
            }
        }

        
        stage('Test SSH Connection') {
            steps {
                script {
                    sshagent([SSH_KEY_CREDENTIALS_ID]) {
                        // Add the test server's SSH key to known_hosts
                        sh "ssh-keyscan -H ${TEST_SERVER_IP} >> ~/.ssh/known_hosts"

                        // Test the SSH connection
                        sh """
                        ssh -o StrictHostKeyChecking=no jenkins@${TEST_SERVER_IP} echo 'SSH connection successful'
                        """
                    }
                }
            }
        }

        stage('Deploy with Ansible') {
            steps {
                script {
                    sshagent(credentials: [SSH_KEY_CREDENTIALS_ID]) {
                        sh "ssh-keyscan -H ${TEST_SERVER_IP} >> ~/.ssh/known_hosts"
                        sh "ssh jenkins@${TEST_SERVER_IP} echo 'SSH connection successful'"
                        sh "ansible-playbook -i ${TEST_SERVER_IP}, ${ANSIBLE_PLAYBOOK} -e docker_image=${DOCKER_IMAGE} -e docker_tag=${DOCKER_TAG} -e aws_region=${AWS_REGION}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
