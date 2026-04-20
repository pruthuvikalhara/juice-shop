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
                        // Simplified config to ensure rule-sets are found
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
                        // Removed the unsupported --connectionTimeout flag
                        sh """
                            ${ODC_HOME}/bin/dependency-check.sh \
                                --project 'Universal-Scan' \
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
                    // FIXED: Replaced ${JOB_NAME} with a hardcoded key "SAST-Pipeline-Project" to avoid space errors
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey='SAST-Pipeline-Project' \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000"
                }
            }
        }

        stage("Quality Gate Enforcement") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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
