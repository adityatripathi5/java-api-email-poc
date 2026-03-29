pipeline {
    agent any
    
    environment {
        // Essential to prevent the OWASP download from timing out again
        NVD_API_KEY = credentials('nvd-api-key') 
    }

    stages {
        stage('Build Java API') {
            steps {
                // Compiles the code to ensure it's valid before scanning
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh '''
                if [ ! -f "gitleaks" ]; then
                    wget -q https://github.com/gitleaks/gitleaks/releases/download/v8.18.1/gitleaks_8.18.1_linux_x64.tar.gz
                    tar -xzf gitleaks_8.18.1_linux_x64.tar.gz
                fi
                # Generates gitleaks-report.json
                ./gitleaks detect -v --report-format json --report-path gitleaks-report.json || true
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                // Generates trivy-results.sarif
                sh 'trivy fs . --format sarif --output trivy-results.sarif || true'
            }
        }

        stage('OWASP Scan') {
            steps {
                // Generates target/dependency-check-report.sarif
                sh 'mvn org.owasp:dependency-check-maven:check -Dformat=SARIF -DfailBuildOnCVSS=11 -DnvdApiKey=$NVD_API_KEY || true'
            }
        }
    }

    post {
        always {
            script {
                echo "Packaging artifacts and sending to Outlook..."
                
                emailext(
                    subject: "🛡️ DevSecOps Scan Reports - POC Build #${env.BUILD_NUMBER}",
                    body: """
                        <h2>Automated DevSecOps Pipeline Execution</h2>
                        <p>The security pipeline has completed its run on the repository.</p>
                        <p><strong>Attached are the raw security reports:</strong></p>
                        <ul>
                            <li>Gitleaks (Secrets) - JSON</li>
                            <li>Trivy (Infrastructure & Vulns) - SARIF</li>
                            <li>OWASP (Dependencies) - SARIF</li>
                        </ul>
                        <p>Please import these into your designated security dashboards.</p>
                    """,
                    to: "aditya.tripathi@opstree.com", 
                    attachmentsPattern: "trivy-results.sarif, gitleaks-report.json, target/dependency-check-report.sarif"
                )
            }
        }
    }
}
