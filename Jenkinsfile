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
                echo "--- STEP 1: CONSOLIDATING DATA ---"
                sh "echo '========================================' > combined-security-report.txt"
                sh "echo 'KING BANANA CONSOLIDATED SECURITY REPORT' >> combined-security-report.txt"
                sh "echo '========================================\n' >> combined-security-report.txt"

                // Add SonarQube
                sh "echo '[SECTION 1: SONARQUBE STATUS]' >> combined-security-report.txt"
                withCredentials([string(credentialsId: 'sonar-token', variable: 'S_TOKEN')]) {
                    sh "curl -u ${S_TOKEN}: -s 'http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project' | jq '.' >> combined-security-report.txt"
                }

                // Add Semgrep
                sh "echo '\n[SECTION 2: SEMGREP SAST FINDINGS]' >> combined-security-report.txt"
                if (fileExists('semgrep-report.txt')) { sh "cat semgrep-report.txt >> combined-security-report.txt" }

                // Add Gitleaks
                sh "echo '\n[SECTION 3: GITLEAKS SECRET SCAN]' >> combined-security-report.txt"
                if (fileExists('gitleaks-report.txt')) { sh "cat gitleaks-report.txt >> combined-security-report.txt" }

                echo "--- STEP 2: SENDING TO AI ---"
                // Read the file content and escape it for JSON
                def reportContent = readFile('combined-security-report.txt').replaceAll('"', '\\\\"').replaceAll('\n', '\\\\n')
                
                withCredentials([string(credentialsId: 'OPENAI_API_KEY', variable: 'OKEY')]) {
                    sh """
                    cat <<EOF > openai-request.json
                    {
                      "model": "gpt-4o-mini",
                      "messages": [{
                        "role": "user", 
                        "content": "Generate a high-end HTML security remediation dashboard for King Banana. Project: Juice Shop. Use the following data: ${reportContent}. Requirements: 1. Dark theme (Slate-900). 2. Highlight the failing Quality Gate and the leaked NVD_API_KEY. 3. Provide specific code fixes for the XSS in navbar.component.html. 4. Use Tailwind CSS. Return ONLY HTML."
                      }],
                      "temperature": 0.2
                    }
                    EOF

                    curl -s https://api.openai.com/v1/chat/completions \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Bearer \$OKEY" \
                      -d @openai-request.json | jq -r '.choices[0].message.content' | sed 's/```html//g' | sed 's/```//g' > security-report.html
                    """
                }
            }
            
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'security-report.html',
                reportName: 'AI Security Remediation'
            ])

            archiveArtifacts artifacts: 'combined-security-report.txt, security-report.html', allowEmptyArchive: true
        }
    }
}
