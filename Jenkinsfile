peline {
    agent any
    
    environment {
        // Get the ThreatMapper API key from Jenkins credentials
        TM_API_KEY = credentials('threatmapper-api-key')
        TM_CONSOLE_URL = "192.168.74.125"
        TM_CONSOLE_PORT = "443"
        DOCKER_IMAGE = "goat-app:${BUILD_ID}"
    }
    
    parameters {
        choice(
            name: 'SEVERITY_THRESHOLD',
            choices: ['none', 'critical', 'high', 'medium'],
            description: 'Minimum severity to flag',
            defaultValue: 'none'
        )
        booleanParam(
            name: 'SECURITY_ALERTS',
            defaultValue: false,
            description: 'Send security alerts to Slack'
        )
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
        
        stage('Security Scan') {
            steps {
                script {
                    // 1. Run ThreatMapper scan
                    def scanOutput = sh(returnStdout: true, script: '''
                        docker run --rm \
                            -e DEEPFENCE_KEY="${TM_API_KEY}" \
                            quay.io/deepfenceio/deepfence_agent_ce:2.5.7 \
                            image scan --image-name ${DOCKER_IMAGE} \
                            --management-console-url ${TM_CONSOLE_URL} \
                            --management-console-port ${TM_CONSOLE_PORT} \
                            --pipeline "${JOB_NAME}" \
                            --json
                    ''')
                    
                    // 2. Parse results
                    def scanResults = readJSON text: scanOutput
                    
                    // 3. Categorize vulnerabilities
                    def vulnsBySeverity = [
                        critical: scanResults.vulnerabilities.count { it.severity == "critical" },
                        high: scanResults.vulnerabilities.count { it.severity == "high" },
                        medium: scanResults.vulnerabilities.count { it.severity == "medium" },
                        low: scanResults.vulnerabilities.count { it.severity == "low" }
                    ]
                    
                    // 4. Generate report
                    def report = """
                    ## ThreatMapper Security Scan Results
                    **Image:** ${DOCKER_IMAGE}
                    **Total Vulnerabilities:** ${scanResults.vulnerabilities.size()}
                    
                    | Severity | Count |
                    |----------|-------|
                    | Critical | ${vulnsBySeverity.critical} |
                    | High     | ${vulnsBySeverity.high} |
                    | Medium   | ${vulnsBySeverity.medium} |
                    | Low      | ${vulnsBySeverity.low} |
                    
                    **Note:** This is a training environment with intentionally vulnerable components.
                    """
                    
                    // 5. Publish report
                    writeFile file: 'threatmapper-report.md', text: report
                    archiveArtifacts artifacts: 'threatmapper-report.md'
                    
                    // 6. Optional: Send to monitoring system
                    if (params.SECURITY_ALERTS && vulnsBySeverity.critical > 0) {
                        slackSend(
                            channel: '#security-alerts',
                            message: "Critical vulnerabilities detected in ${JOB_NAME} - ${vulnsBySeverity.critical} critical issues"
                        )
                    }
                    
                    // 7. Custom threshold check
                    if (params.SEVERITY_THRESHOLD == 'critical' && vulnsBySeverity.critical > 0) {
                        unstable("Critical vulnerabilities found - marking build as unstable")
                    }
                    else if (params.SEVERITY_THRESHOLD == 'high' && (vulnsBySeverity.critical > 0 || vulnsBySeverity.high > 0)) {
                        unstable("Critical/High vulnerabilities found - marking build as unstable")
                    }
                }
            }
        }
        
        stage('Deploy to App VM') {
            steps {
                script {
                    // Deploy the built image to your application VM
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
            
            // Archive reports if they exist
            archiveArtifacts artifacts: 'threatmapper-report.md', allowEmptyArchive: true
        }
    }
}
