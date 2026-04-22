pipeline {
    agent any
    
    triggers {
        githubPush()
    }

    environment {
        SCANNER_HOME = '/opt/sonar-scanner'
        ODC_HOME = '/opt/dependency-check'
        NVD_API_KEY = '8613479c-ad4f-4ed9-b39f-7346f723a600'
    }

    stages {
        stage('Universal Checkout') {
            steps { checkout scm }
        }

        stage('Multi-Tool Security Scan') {
            parallel {
                stage('Secret Scan') {
                    steps { sh 'gitleaks detect --source . --verbose --redact || true' }
                }
                stage('SAST (Pro Semgrep)') {
                    steps {
                        sh 'semgrep scan --config p/security-audit --config p/javascript --config p/nodejsscan --text --output=semgrep-report.txt || true'
                    }
                }
                stage('SCA (Dependency Check)') {
                    steps {
                        sh "${ODC_HOME}/bin/dependency-check.sh --project 'Juice-Shop' --scan . --format 'ALL' --out . --nvdApiKey ${NVD_API_KEY} || true"
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey='SAST-Pipeline-Project' -Dsonar.sources=. -Dsonar.host.url=http://localhost:9000"
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        // Small sleep to ensure SonarQube API updates its database
                        sh 'sleep 5' 
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "--- Capturing Final Security Intel ---"
                
                // 1. Better Data Extraction (Case-Insensitive 'High' or 'Critical')
                def semgrepRaw = fileExists('semgrep-report.txt') ? sh(script: "grep -iE 'high|critical|error' semgrep-report.txt | head -n 15", returnStdout: true).trim() : "No report found."
                
                // 2. Capture SonarQube Status safely
                def sonarRaw = sh(script: "curl -s http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project | jq -r '.projectStatus.status'", returnStdout: true).trim()

                // 3. The Professional Prompt
                def reportPrompt = """
                Generate a professional HTML security dashboard for project 'Juice Shop'.
                Current Status: SonarQube is ${sonarRaw}.
                Findings: ${semgrepRaw}.
                
                Requirements:
                - Use Tailwind CSS CDN.
                - Theme: Slate-900 background.
                - Sections: Dashboard Summary, Critical Vulnerabilities Table, and Remediation Code.
                - Use professional badges (Red for Critical, Orange for High).
                - Return ONLY the <html> code. No markdown.
                """.replaceAll(/["\\]/, '')

                withCredentials([string(credentialsId: 'GEMINI_API_KEY', variable: 'AI_KEY')]) {
                    sh """
                    curl -s -X POST 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${AI_KEY}' \
                         -H 'Content-Type: application/json' \
                         -d '{"contents": [{"parts":[{"text": "${reportPrompt}"}]}]}' \
                         | jq -r '.candidates[0].content.parts[0].text' \
                         | sed 's/```html//g' | sed 's/```//g' > security-report.html
                    """
                }
            }
            
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'security-report.html',
                reportName: 'AI Security Analysis'
            ])

            archiveArtifacts artifacts: '*.txt, *.html, *.json', allowEmptyArchive: true
            cleanWs()
        }
    }
}
