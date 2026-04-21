pipeline {
    agent any
    environment {
        SCANNER_HOME = '/opt/sonar-scanner'
        ODC_HOME = '/opt/dependency-check'
        NVD_API_KEY = '8613479c-ad4f-4ed9-b39f-7346f723a600'
    }
    stages {
        stage('Universal Checkout') {
            steps { 
                checkout scm 
            }
        }

        stage('Multi-Tool Security Scan') {
            parallel {
                stage('Secret Scan') {
                    steps {
                        sh 'gitleaks detect --source . --verbose --redact || true'
                    }
                }
                stage('SAST (Pro Semgrep)') {
                    steps {
                        sh '''
                            semgrep scan \
                                --config p/security-audit \
                                --config p/javascript \
                                --config p/nodejsscan \
                                --text \
                                --metrics=off \
                                --output=semgrep-report.txt \
                                --error || true
                        '''
                    }
                }
                stage('SCA (Dependency Check)') {
                    steps {
                        sh """
                            ${ODC_HOME}/bin/dependency-check.sh \
                                --project 'Juice-Shop-Analysis' \
                                --scan . \
                                --format 'ALL' \
                                --out . \
                                --nvdApiKey ${NVD_API_KEY} \
                                --nvdApiDelay 10000 || true
                        """
                    }
                }
            }
        }

        stage('SonarQube Global Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    // Using a distinct project key for Juice Shop to keep results clean
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey='SAST-Pipeline-Project' \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000"
                }
            }
        }

        stage("Quality Gate Enforcement") {
            steps {
                // Increased timeout to ensure Juice Shop background processing completes
                timeout(time: 15, unit: 'MINUTES') {
                    script {
                        // This waits for the Webhook from SonarQube
                        def qg = waitForQualityGate()
                        
                        // FAIL LOGIC: If status is not 'OK', the pipeline stops here
                        if (qg.status != 'OK') {
                            error "PIPELINE ABORTED: Security standards not met. Status: ${qg.status}. Check: http://localhost:9000"
                        } else {
                            echo "SUCCESS: Code meets professional security standards."
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'semgrep-report.txt, dependency-check-report.html, *.json, *.txt', allowEmptyArchive: true
        }
    }
}
