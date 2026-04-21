pipeline {
    agent any
    
    triggers {
        // Automatically start the build when you push to GitHub
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
                        // Detects leaked keys/passwords
                        sh 'gitleaks detect --source . --verbose --redact || true'
                    }
                }
                stage('SAST (Pro Semgrep)') {
                    steps {
                        // Static analysis for code vulnerabilities
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
                        // Scans third-party libraries for known CVEs
                        sh """
                            ${ODC_HOME}/bin/dependency-check.sh \
                                --project 'Juice-Shop-Analysis' \
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
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey='SAST-Pipeline-Project' \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000"
                }
            }
        }

        stage("Quality Gate Enforcement") {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    script {
                        // Waits for SonarQube to finish processing Overall Code conditions
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "PIPELINE ABORTED: Security standards not met. Status: ${qg.status}. Vulnerabilities detected!"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // --- AI SUMMARIZATION PART ---
                if (fileExists('semgrep-report.txt')) {
                    echo "--- Consulting AI Security Analyst for Final Outcome ---"
                    
                    // Pulling the most relevant 40 lines of the report
                    def reportSnippet = sh(script: "head -n 40 semgrep-report.txt", returnStdout: true).trim()
                    
                    // Wrapping in try-catch so an AI error doesn't break artifact archiving
                    try {
                        withCredentials([string(credentialsId: 'GEMINI_API_KEY', variable: 'AI_KEY')]) {
                            def response = sh(
                                script: """
                                curl -s -X POST 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${AI_KEY}' \
                                     -H 'Content-Type: application/json' \
                                     -d '{
                                       "contents": [{
                                         "parts":[{"text": "You are a DevSecOps expert. Summarize these results for King Banana. The pipeline failed. Explain the top 3 risks found in the code and a simple fix. Results: ${reportSnippet.replaceAll(/["\\]/, '')}"}]
                                       }]
                                     }'
                                """,
                                returnStdout: true
                            )
                            echo "========================================================="
                            echo "                🤖 AI SECURITY OUTCOME                  "
                            echo "========================================================="
                            echo response
                            echo "========================================================="
                        }
                    } catch (Exception e) {
                        echo "AI Summary failed: ${e.message}"
                    }
                }
            }
            
            // Archive the findings before the workspace is wiped
            archiveArtifacts artifacts: 'semgrep-report.txt, dependency-check-report.html, *.json, *.txt', allowEmptyArchive: true
            
            // Cleanup workspace to keep the Ubuntu server clean
            cleanWs()
        }
    }
}
