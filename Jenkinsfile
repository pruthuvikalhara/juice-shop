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
        // Note: Move secrets like NVD_API_KEY to Jenkins Credentials for better security later
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
                echo "--- STEP 1: CONSOLIDATING ALL SECURITY DATA ---"
                
                // Initialize the master report file
                sh """
                echo '========================================' > combined-security-report.txt
                echo 'KING BANANA MASTER SECURITY REPORT' >> combined-security-report.txt
                echo 'Generated: \$(date)' >> combined-security-report.txt
                echo '========================================\n' >> combined-security-report.txt
                """

                // 1. Pull SonarQube Quality Gate Status
                sh "echo '[SECTION 1: SONARQUBE METRICS]' >> combined-security-report.txt"
                try {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'S_TOKEN')]) {
                        sh "curl -u ${S_TOKEN}: -s 'http://localhost:9000/api/qualitygates/project_status?projectKey=SAST-Pipeline-Project' | jq '.' >> combined-security-report.txt"
                    }
                } catch (Exception e) {
                    sh "echo 'SonarQube data could not be retrieved.' >> combined-security-report.txt"
                }

                // 2. Append Semgrep Findings
                sh "echo '\n[SECTION 2: SEMGREP SAST FINDINGS]' >> combined-security-report.txt"
                if (fileExists('semgrep-report.txt')) {
                    sh "cat semgrep-report.txt >> combined-security-report.txt"
                }

                // 3. Append Gitleaks Findings
                sh "echo '\n[SECTION 3: GITLEAKS SECRET SCAN]' >> combined-security-report.txt"
                if (fileExists('gitleaks-report.txt')) {
                    sh "cat gitleaks-report.txt >> combined-security-report.txt"
                }

                echo "--- STEP 2: GENERATING AI REMEDIATION DASHBOARD ---"
                
                // Sanitize content for JSON payload
                def reportContent = readFile('combined-security-report.txt').replaceAll('"', '\\\\"').replaceAll('\n', '\\\\n')
                
                withCredentials([string(credentialsId: 'OPENAI_API_KEY', variable: 'OKEY')]) {
                    // Create the JSON request file
                    sh """
                    cat <<EOF > openai-request.json
                    {
                      "model": "gpt-4o-mini",
                      "messages": [{
                        "role": "user", 
                        "content": "You are a Senior DevSecOps Engineer. Analyze this report for 'King Banana': ${reportContent}. Create a professional, vibrant HTML dashboard using Tailwind CSS (Dark Mode). Include: 1. Executive Summary. 2. Critical Vulnerability Alerts. 3. Exact code fixes for XSS and Hardcoded Secrets. 4. Strategic recommendations for the Jenkins pipeline. Return ONLY HTML."
                      }],
                      "temperature": 0.3
                    }
                    EOF
                    """

                    // Execute API call and save raw response
                    sh 'curl -s https://api.openai.com/v1/chat/completions -H "Content-Type: application/json" -H "Authorization: Bearer \$OKEY" -d @openai-request.json | jq -r ".choices[0].message.content" > raw_ai_output.txt'

                    // Clean the HTML from markdown code blocks safely
                    sh "sed 's/```html//g' raw_ai_output.txt > temp_clean.html"
                    sh "sed 's/```//g' temp_clean.html > security-report.html"
                }
            }
            
            // Publish the visual dashboard
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'security-report.html',
                reportName: 'AI Security Remediation'
            ])

            // Archive the raw combined report and the final HTML for history
            archiveArtifacts artifacts: 'combined-security-report.txt, security-report.html', allowEmptyArchive: true
            
            echo "--- PIPELINE COMPLETE: CHECK THE AI SECURITY REMEDIATION TAB ---"
        }
    }
}
