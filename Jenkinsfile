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
                        sh 'sleep 5' 
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "--- Starting OpenAI Security Analysis ---"
                
                // 1. Collect findings from the scanners
                def semgrepRaw = fileExists('semgrep-report.txt') ? sh(script: "grep -iE 'high|critical|error' semgrep-report.txt | head -n 15", returnStdout: true).trim() : "No high-risk issues."
                def sonarRaw = sh(script: "curl -s http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project | jq -r '.projectStatus.status'", returnStdout: true).trim()

                // 2. The Prompt for OpenAI
                def reportPrompt = """
                Generate a professional HTML security dashboard for 'King Banana'.
                SonarQube Status: ${sonarRaw}
                Semgrep Issues: ${semgrepRaw}
                
                Requirements:
                - Use Tailwind CSS (CDN).
                - Dark Theme (Slate-900).
                - Sections: Dashboard Summary, Risk Table, and Detailed Fixes.
                - Return ONLY the raw HTML code. Do not use markdown.
                """.replaceAll(/["\\]/, '')

                // 3. The OpenAI API Call
                withCredentials([string(credentialsId: 'OPENAI_API_KEY', variable: 'OPENAI_KEY')]) {
                    sh """
                    curl -s https://api.openai.com/v1/chat/completions \
                      -H "Content-Type: application/json" \
                      -H "Authorization: Bearer ${OPENAI_KEY}" \
                      -d '{
                        "model": "gpt-4o-mini",
                        "messages": [{"role": "user", "content": "${reportPrompt}"}],
                        "temperature": 0.7
                      }' | jq -r '.choices[0].message.content' \
                         | sed 's/```html//g' | sed 's/```//g' > security-report.html
                    """
                }
            }
            
            // 4. Publish to the Jenkins Sidebar
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
