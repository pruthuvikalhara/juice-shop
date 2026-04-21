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
                        sh 'semgrep scan --config p/security-audit --config p/javascript --config p/nodejsscan --text --metrics=off --output=semgrep-report.txt --error || true'
                    }
                }
                stage('SCA (Dependency Check)') {
                    steps {
                        sh """
                            ${ODC_HOME}/bin/dependency-check.sh --project 'Juice-Shop' --scan . --format 'ALL' --out . --nvdApiKey ${NVD_API_KEY} || true
                        """
                    }
                }
            }
        }

        stage('SonarQube Global Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey='SAST-Pipeline-Project' -Dsonar.sources=. -Dsonar.host.url=http://localhost:9000"
                }
            }
        }

        stage("Quality Gate Enforcement") {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            echo "Quality Gate Failed, but we will generate the AI report before stopping."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "--- Generating High-End AI Security Dashboard ---"
                
                // 1. Gather Context
                def semgrepRaw = fileExists('semgrep-report.txt') ? sh(script: "grep 'HIGH' semgrep-report.txt | head -n 20", returnStdout: true).trim() : "No critical issues."
                def sonarRaw = sh(script: "curl -s http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project", returnStdout: true).trim()

                // 2. The Pro-UI Prompt
                def reportPrompt = """
                Generate a professional, single-file HTML security audit dashboard for 'King Banana'.
                Data: SonarQube Status: ${sonarRaw}, Semgrep High Risks: ${semgrepRaw}.
                
                Requirements:
                - Use Tailwind CSS: <script src='https://cdn.tailwindcss.com'></script>
                - Theme: Slate-900 background, modern SaaS look.
                - Sections: Overall Security Grade (A-F), Critical Findings Table, and Actionable Remediation.
                - Table: Columns for Vulnerability, Impact, and Fix. Use glowing red/orange badges.
                - Tone: Professional, elite security auditor.
                - Return ONLY the HTML code.
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
            
            // 3. Publish the modern dashboard
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'security-report.html',
                reportName: 'AI Security Remediation'
            ])

            archiveArtifacts artifacts: 'semgrep-report.txt, dependency-check-report.html, *.json, *.txt', allowEmptyArchive: true
            cleanWs()
        }
    }
}
