pipeline {
    agent any
    
    triggers {
        // This tells Jenkins to listen for the GitHub Webhook "push" event
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
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "PIPELINE ABORTED: Security standards not met. Status: ${qg.status}."
                        } else {
                            echo "SUCCESS: Code meets professional security standards."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'semgrep-report.txt, dependency-check-report.html, *.json, *.txt', allowEmptyArchive: true
            // Best practice: Clean up the workspace to save disk space on your Ubuntu server
            cleanWs()
        }
    }
}
