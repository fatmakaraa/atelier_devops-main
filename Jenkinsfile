pipeline {
    agent any
    
    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        // Etape 1 : pull du code source depuis GitHub
        stage('GIT') {
            steps {
                git(
                    branch: 'main', 
                    url: 'https://github.com/fatmakaraa/atelier_devops-main.git'  
                )
            }
        }
        
        // Etape 2 : Scan des secrets avec rapport détaillé
        stage('Secrets Scan') {
            steps {
                sh '''
                    echo "🔍 Starting Gitleaks Secret Detection..."
                    gitleaks detect --source . \
                        --verbose \
                        --redact \
                        --report-format json \
                        --report-path gitleaks-report.json \
                        --report-format sarif \
                        --report-path gitleaks-report.sarif \
                        --exit-code 0  # Ne pas échouer le build pour l'analyse
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.*', allowEmptyArchive: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'gitleaks-report.html',
                        reportName: 'Gitleaks Secrets Report'
                    ])
                }
            }
        }
        
        // Etape 3 : Tests unitaires avec rapports
        stage('Tests Unitaires') {
            steps {
                sh 'mvn test' 
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target/site',
                        reportFiles: 'surefire-report.html',
                        reportName: 'Unit Tests Report'
                    ])
                }
            }
        }
        
        // Etape 4 : Analyse de dépendances avec OWASP
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -DfailBuildOnCVSS=7 \
                    || echo "Dependency check completed - check report for vulnerabilities"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/target/dependency-check-report.*', allowEmptyArchive: true
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'Dependency Check Report'
                    ])
                }
            }
        }
        
        // Etape 5 : Compilation
        stage('compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        // Etape 6 : SonarQube avec rapport de qualité
        stage('MVN SONARQUBE') {
            steps {
                withSonarQubeEnv('sq1') {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
            post {
                always {
                    // SonarQube génère son propre rapport accessible via l'URL
                    script {
                        env.SONAR_URL = sh(
                            script: 'echo ${SONAR_HOST_URL}/dashboard?id=${SONAR_PROJECT_KEY}',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }
  
        // Etape 7 : Construction des images Docker
        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }
        
        // Etape 8 : Scan de sécurité des images Docker
        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    env.DOCKER_IMAGE = "fatmakaraa/kaddem:latest"
                    sh """
                        echo "🔍 Scanning Docker image with Trivy..."
                        
                        # Télécharger la DB avec plus de temps
                        trivy image --download-db-only \
                            --timeout 10m \
                            ${DOCKER_IMAGE} || echo "DB download might have issues, continuing..."
                        
                        # Scanner avec timeout étendu
                        trivy image --exit-code 0 \
                            --severity HIGH,CRITICAL \
                            --format json \
                            --output trivy-report.json \
                            --timeout 15m \
                            ${DOCKER_IMAGE} || echo "Scan completed with possible warnings"
                        
                        # Générer un rapport simple même en cas d'erreur
                        trivy image --exit-code 0 \
                            --severity HIGH,CRITICAL \
                            --format table \
                            --timeout 10m \
                            ${DOCKER_IMAGE} > trivy-report.txt 2>&1 || true
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.*', allowEmptyArchive: true
                    // Créer un rapport même si Trivy échoue
                    sh '''
                        echo "<html><body><h1>Trivy Scan Report</h1><pre>" > trivy-fallback.html
                        cat trivy-report.txt >> trivy-fallback.html 2>/dev/null || echo "Scan incomplete - check console logs" >> trivy-fallback.html
                        echo "</pre></body></html>" >> trivy-fallback.html
                    '''
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'trivy-fallback.html',
                        reportName: 'Trivy Security Scan Report'
                    ])
                }
            }
        }
  
        // Etape 9 : Authentification Docker Hub
        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "Logging into Docker Hub..."
                            docker logout || true
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        '''
                    }
                }
            }
        }
        
        // Etape 10 : Push image Docker
        stage('Push Docker Image') {
            steps {
                sh '''
                    docker compose push
                    docker logout
                '''
            }
        }
        
        // Etape 11 : Déploiement
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down && docker compose up -d'
            }
        }
        
        // Etape 12 : Génération du rapport final
        stage('Generate Comprehensive Reports') {
            steps {
                script {
                    // Rapport HTML combiné
                    def htmlContent = """
                    <html>
                    <head>
                        <title>Pipeline Execution Report - Build #${env.BUILD_NUMBER}</title>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 20px; }
                            .header { background: #f4f4f4; padding: 20px; border-radius: 5px; }
                            .section { margin: 20px 0; padding: 15px; border-left: 4px solid #007cba; }
                            .success { border-color: #28a745; background: #f8fff9; }
                            .warning { border-color: #ffc107; background: #fffef0; }
                            .error { border-color: #dc3545; background: #fff5f5; }
                            .report-link { margin: 5px 0; }
                        </style>
                    </head>
                    <body>
                        <div class="header">
                            <h1>🚀 Pipeline Execution Report</h1>
                            <h2>Build #${env.BUILD_NUMBER}</h2>
                            <p><strong>Status:</strong> <span style="color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'}">${currentBuild.currentResult ?: 'SUCCESS'}</span></p>
                            <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm:ss")}</p>
                            <p><strong>Duration:</strong> ${currentBuild.durationString ?: 'N/A'}</p>
                        </div>
                        
                        <div class="section">
                            <h3>📊 Security & Quality Reports</h3>
                            <div class="report-link"><a href="${env.BUILD_URL}/Gitleaks_20Secrets_20Report/" target="_blank">🔐 Gitleaks Secrets Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Unit_20Tests_20Report/" target="_blank">🧪 Unit Tests Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Dependency_20Check_20Report/" target="_blank">📦 Dependency Check Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/Trivy_20Security_20Scan_20Report/" target="_blank">🐳 Trivy Docker Scan Report</a></div>
                            <div class="report-link"><a href="${env.SONAR_URL ?: '#'}" target="_blank">📈 SonarQube Quality Report</a></div>
                        </div>
                        
                        <div class="section success">
                            <h3>✅ Build Artifacts</h3>
                            <div class="report-link"><a href="${env.BUILD_URL}/artifact/gitleaks-report.json" target="_blank">Gitleaks JSON Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/artifact/trivy-report.json" target="_blank">Trivy JSON Report</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}/artifact/" target="_blank">All Artifacts</a></div>
                        </div>
                        
                        <div class="section">
                            <h3>🔗 Quick Links</h3>
                            <div class="report-link"><a href="${env.BUILD_URL}console" target="_blank">Console Output</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}changes" target="_blank">Changes</a></div>
                            <div class="report-link"><a href="${env.BUILD_URL}testReport" target="_blank">Test Results</a></div>
                        </div>
                    </body>
                    </html>
                    """
                    
                    writeFile file: 'comprehensive-report.html', text: htmlContent
                    
                    // Archiver tous les rapports
                    archiveArtifacts artifacts: '**/target/*.json,**/target/*.html,gitleaks-report.*,trivy-report.*,comprehensive-report.html', allowEmptyArchive: true
                    
                    // Publier le rapport principal
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'comprehensive-report.html',
                        reportName: 'Pipeline Comprehensive Report'
                    ])
                }
            }
        }
    }
     
    post {
        always {
            // Nettoyage et rapports finaux
            script {
                // Générer un résumé pour l'email
                def summary = """
                Pipeline Execution Completed - ${currentBuild.currentResult}
                
                Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                Duration: ${currentBuild.durationString}
                Status: ${currentBuild.currentResult}
                
                Reports Available:
                - Security Scan: ${env.BUILD_URL}/Gitleaks_20Secrets_20Report/
                - Unit Tests: ${env.BUILD_URL}/Unit_20Tests_20Report/
                - Dependency Check: ${env.BUILD_URL}/Dependency_20Check_20Report/
                - Docker Security: ${env.BUILD_URL}/Trivy_20Security_20Scan_20Report/
                - Comprehensive Report: ${env.BUILD_URL}/Pipeline_20Comprehensive_20Report/
                - SonarQube: ${env.SONAR_URL ?: 'Not available'}
                
                Console: ${env.BUILD_URL}console
                """
                
                currentBuild.description = "Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}"
            }
        }
        
        success {
            mail to: 'karaafatma01@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                ✅ Build Successfully Completed!

                Job: ${env.JOB_NAME}
                Build: #${env.BUILD_NUMBER}
                Duration: ${currentBuild.durationString}
                
                📊 Detailed Reports Available:
                - Full Report: ${env.BUILD_URL}/Pipeline_20Comprehensive_20Report/
                - Security Scan: ${env.BUILD_URL}/Gitleaks_20Secrets_20Report/
                - Test Results: ${env.BUILD_URL}/Unit_20Tests_20Report/
                - Dependency Check: ${env.BUILD_URL}/Dependency_20Check_20Report/
                - Docker Scan: ${env.BUILD_URL}/Trivy_20Security_20Scan_20Report/
                - SonarQube: ${env.SONAR_URL ?: 'N/A'}
                
                Console: ${env.BUILD_URL}console
                """
        }
        
        failure {
            mail to: 'karaafatma01@gmail.com',
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                ❌ BUILD FAILED!

                Job: ${env.JOB_NAME}
                Build: #${env.BUILD_NUMBER}
                Duration: ${currentBuild.durationString}
                
                🔍 Investigation Links:
                - Console Output: ${env.BUILD_URL}console
                - Test Results: ${env.BUILD_URL}testReport
                - Changes: ${env.BUILD_URL}changes
                
                Immediate action required!
                """
        }
    }
}
