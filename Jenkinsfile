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
                        // We use --text so grep can read it easily later
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
                        // Sleep ensures the SonarQube Database is updated before we curl the API
                        sh 'sleep 10' 
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "--- DATA COLLECTION DEBUG ---"
                
                // 1. Capture SonarQube Status with Fallback
                def sonarStatus = sh(script: "curl -s http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project | jq -r '.projectStatus.status' || echo 'PENDING'", returnStdout: true).trim()
                echo "SonarQube Debug: Captured status is [${sonarStatus}]"

                // 2. Capture Semgrep Findings with Fallback
                def semgrepData = "No high-risk issues found."
                if (fileExists('semgrep-report.txt')) {
                    semgrepData = sh(script: "grep -iE 'high|critical|error|vulnerability' semgrep-report.txt | head -n 20", returnStdout: true).trim()
                    if (semgrepData == "") { semgrepData = "Scan complete: No Critical/High issues detected in logs." }
                }
                echo "Semgrep Debug: Captured findings: ${semgrepData}"

                // 3. Construct the OpenAI Prompt
                def reportPrompt = """
                Generate a professional HTML security dashboard for King Banana.
                Project: Juice Shop
                SonarQube Status: ${sonarStatus}
                Semgrep Findings: ${semgrepData}
                
                Requirements:
                - Use Tailwind CSS (CDN).
                - Dark Theme (Slate-900).
                - Sections: Security Summary, Risk Table, and Remediation Plan.
                - Return ONLY raw HTML code. No markdown or backticks.
                """.replaceAll(/["\\]/, '')

                // 4. OpenAI API Call
                withCredentials([string(credentialsId: 'OPENAI_API_KEY', variable: 'OPENAI_KEY')]) {
                    sh """
                    curl -s https://api.openai.com/v1/chat/completions \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Bearer ${OPENAI_KEY}" \
                      -d '{
                        "model": "gpt-4o-mini",
                        "messages": [{"role": "user", "content": "${reportPrompt}"}],
                        "temperature": 0.2
                      }' | jq -r '.choices[0].message.content' \
                         | sed 's/```html//g' | sed 's/```//g' > security-report.html
                    """
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

            archiveArtifacts artifacts: '*.txt, *.html, *.json', allowEmptyArchive: true
            cleanWs()
        }
    }
}
