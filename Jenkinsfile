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

        stage('Security Scans (Skipped for Testing)') {
            steps {
                script {
                    echo "Skipping real scans to save time..."
                    // We create dummy files so the 'post' block has something to read
                    sh "echo 'DUMMY SEMGREP DATA: XSS Found in navbar.html' > semgrep-report.txt"
                    sh "echo 'DUMMY GITLEAKS DATA: Secret found in Jenkinsfile' > gitleaks-report.txt"
                }
            }
        }

        stage('SonarQube (Skipped for Testing)') {
            steps {
                echo "Skipping SonarQube analysis..."
                // We don't need to do anything here; the 'post' block will 
                // still try to curl the API to get the last known status.
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

                // Add SonarQube (This will still pull the last result from your server)
                sh "echo '[SECTION 1: SONARQUBE STATUS]' >> combined-security-report.txt"
                try {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'S_TOKEN')]) {
                        sh "curl -u ${S_TOKEN}: -s 'http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project' | jq '.' >> combined-security-report.txt"
                    }
                } catch (Exception e) {
                    sh "echo 'Failed to fetch SonarQube data' >> combined-security-report.txt"
                }

                // Add Semgrep (Using the dummy file we created above)
                sh "echo '\n[SECTION 2: SEMGREP SAST FINDINGS]' >> combined-security-report.txt"
                if (fileExists('semgrep-report.txt')) { sh "cat semgrep-report.txt >> combined-security-report.txt" }

                // Add Gitleaks (Using the dummy file we created above)
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
                        "content": "Generate a high-end HTML security remediation dashboard for King Banana. Project: Juice Shop. Use the following data: ${reportContent}. Requirements: 1. Dark theme (Slate-900). 2. Highlight failing metrics. 3. Provide code fixes. 4. Use Tailwind CSS. Return ONLY HTML."
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
