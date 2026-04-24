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
                        sh 'gitleaks detect --source . --verbose --redact || true'
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
                echo "--- STARTING DATA COLLECTION FOR AI ---"
                
                def sonarStatus = "UNKNOWN"
                
                // 1. Capture SonarQube Status using your 'sonar-token'
                try {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'S_TOKEN')]) {
                        sonarStatus = sh(script: "curl -u ${S_TOKEN}: -s 'http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project' | jq -r '.projectStatus.status'", returnStdout: true).trim()
                    }
                } catch (Exception e) { 
                    echo "Failed to fetch Sonar status: ${e.message}"
                    sonarStatus = "FETCH_ERROR" 
                }
                echo "SonarQube Debug: Captured status is [${sonarStatus}]"

                // 2. Capture Semgrep Findings and strip quotes to prevent JSON breakage
                def semgrepData = "No critical issues detected."
                if (fileExists('semgrep-report.txt')) {
                    // This strips out double quotes so they don't break the OpenAI JSON structure
                    semgrepData = sh(script: "grep -iE 'high|critical|error' semgrep-report.txt | head -n 10 | tr -d '\"' | tr -d \"'\" | tr -d '\n' ", returnStdout: true).trim()
                    if (semgrepData == "") { semgrepData = "Scan complete: No high-priority findings." }
                }
                echo "Semgrep Debug: Captured findings: ${semgrepData}"

                // 3. Call OpenAI using a clean JSON file (Best Practice for SysAdmins)
                withCredentials([string(credentialsId: 'OPENAI_API_KEY', variable: 'OKEY')]) {
                    sh """
                    # Create the JSON payload file safely
                    cat <<EOF > openai-request.json
                    {
                      "model": "gpt-4o-mini",
                      "messages": [
                        {
                          "role": "user", 
                          "content": "Generate a professional HTML security dashboard for King Banana. Project: Juice Shop. SonarQube Status: ${sonarStatus}. Findings: ${semgrepData}. Use Tailwind CSS. Dark theme. Sections: Risk Summary, Table of Issues, and Remediation. Return ONLY HTML. No markdown."
                        }
                      ],
                      "temperature": 0.2
                    }
                    EOF

                    # Send the file to OpenAI
                    curl -s https://api.openai.com/v1/chat/completions \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Bearer \$OKEY" \
                      -d @openai-request.json | jq -r '.choices[0].message.content' \
                      | sed 's/```html//g' | sed 's/```//g' > security-report.html
                    """
                }

                // 4. Final Fail-Safe: If file is empty or 'null', create a basic message
                def reportContent = readFile('security-report.html').trim()
                if (reportContent == "null" || reportContent == "" || !fileExists('security-report.html')) {
                    sh "echo '<html><body style=\"background:#1a202c;color:white;padding:50px;font-family:sans-serif;\"><h1>AI Analysis Unavailable</h1><p>OpenAI returned no data. Check your API usage/credits.</p></body></html>' > security-report.html"
                }
            }
            
            // 5. Publish to Jenkins Sidebar
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'security-report.html',
                reportName: 'AI Security Remediation'
            ])

            archiveArtifacts artifacts: 'security-report.html, openai-request.json, *.txt', allowEmptyArchive: true
            cleanWs()
        }
    }
}
