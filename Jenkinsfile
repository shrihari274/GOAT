peline {
    agent any
    
    environment {
        // Get the ThreatMapper API key from Jenkins credentials
        TM_API_KEY = credentials('threatmapper-api-key')
        TM_CONSOLE_URL = "192.168.74.125"
        TM_CONSOLE_PORT = "443"
        DOCKER_IMAGE = "goat-app:${BUILD_ID}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/shrihari274/GOAT'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the GOAT application Docker image
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                }
            }
        }
        
        stage('Run ThreatMapper Scan') {
            steps {
                script {
                    // Run ThreatMapper scan on the built image
                    sh '''
                    docker run --rm \
                        -e DEEPFENCE_KEY="${TM_API_KEY}" \
                        quay.io/deepfenceio/deepfence_agent_ce:2.5.7 \
                        image scan --image-name ${DOCKER_IMAGE} \
                        --management-console-url ${TM_CONSOLE_URL} \
                        --management-console-port ${TM_CONSOLE_PORT} \
                        --pipeline "${JOB_NAME}"
                    '''
                    
                    // Alternatively, scan the running container (after deployment)
                    // sh '''
                    // docker run --rm \
                    //     -e DEEPFENCE_KEY="${TM_API_KEY}" \
                    //     quay.io/deepfenceio/deepfence_agent_ce:2.5.7 \
                    //     scan --target <container-ip> \
                    //     --management-console-url ${TM_CONSOLE_URL} \
                    //     --management-console-port ${TM_CONSOLE_PORT} \
                    //     --pipeline "${JOB_NAME}"
                    // '''
                }
            }
        }
        
        stage('Deploy to App VM') {
            steps {
                script {
                    // Deploy the built image to your application VM
                    // This requires SSH access to your App VM
                    sshagent(['your-ssh-credential-id']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no shree@192.168.74.127 \
                            "docker stop goat-app || true && \
                             docker rm goat-app || true && \
                             docker run -d --name goat-app -p 80:3000 ${DOCKER_IMAGE}"
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean up
            sh 'docker rmi ${DOCKER_IMAGE} || true'
        }
    }
}
