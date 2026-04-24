pipeline {
    agent any
    
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    environment {
        SCANNER_HOME = '/opt/sonar-scanner'
        ODC_HOME = '/opt/dependency-check'
        NVD_API_KEY = '8613479c-ad4f-4ed9-b39f-7346f723a600'
    }

    stages {
        stage('Checkout & Cleanup') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Semgrep SAST') {
                    steps {
                        sh 'semgrep scan --config p/security-audit --text --output=semgrep-report.txt || true'
                    }
                }
                stage('Gitleaks Secrets') {
                    steps {
                        // We redirect output to a file so we can capture it later
                        sh 'gitleaks detect --source . --verbose --redact > gitleaks-report.txt || true'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey='SAST-Pipeline-Project' -Dsonar.sources=."
                }
            }
        }

        stage("Quality Gate Wait") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        waitForQualityGate()
                        sh 'sleep 10' 
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "--- CONSOLIDATING ALL SECURITY FINDINGS ---"
                
                // 1. Initialize the combined file
                sh "echo '========================================' > combined-security-report.txt"
                sh "echo 'KING BANANA CONSOLIDATED SECURITY REPORT' >> combined-security-report.txt"
                sh "echo 'Generated: \$(date)' >> combined-security-report.txt"
                sh "echo '========================================\n' >> combined-security-report.txt"

                // 2. Append SonarQube Results (The API Pull)
                sh "echo '[SECTION 1: SONARQUBE STATUS]' >> combined-security-report.txt"
                try {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'S_TOKEN')]) {
                        sh "curl -u ${S_TOKEN}: -s 'http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project' | jq '.' >> combined-security-report.txt"
                    }
                } catch (Exception e) {
                    sh "echo 'Failed to fetch SonarQube data' >> combined-security-report.txt"
                }
                sh "echo '\n' >> combined-security-report.txt"

                // 3. Append Semgrep Findings
                sh "echo '[SECTION 2: SEMGREP SAST FINDINGS]' >> combined-security-report.txt"
                if (fileExists('semgrep-report.txt')) {
                    sh "cat semgrep-report.txt >> combined-security-report.txt"
                } else {
                    sh "echo 'Semgrep report not found.' >> combined-security-report.txt"
                }
                sh "echo '\n' >> combined-security-report.txt"

                // 4. Append Gitleaks Findings
                sh "echo '[SECTION 3: GITLEAKS SECRET SCAN]' >> combined-security-report.txt"
                if (fileExists('gitleaks-report.txt')) {
                    sh "cat gitleaks-report.txt >> combined-security-report.txt"
                } else {
                    sh "echo 'No Gitleaks report found or no secrets detected.' >> combined-security-report.txt"
                }

                // 5. Final Print to Console (For quick viewing)
                sh "cat combined-security-report.txt"
            }
            
            // Archive the file so it appears in the Jenkins UI
            archiveArtifacts artifacts: 'combined-security-report.txt, *.txt', allowEmptyArchive: true
            cleanWs()
        }
    }
}
