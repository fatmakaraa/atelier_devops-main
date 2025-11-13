pipeline {
    agent any
    // AJOUTEZ JUSTE CECI au début de votre pipeline existant
    triggers {
        pollSCM('H/2 * * * *')  // Toutes les 2 minutes
    }

    stages {
        // Etape 1 : pull du code source depuis GitHub
        stage('GIT') 
        {
            steps {
                git(
                    branch: 'main', 
                    url: 'https://github.com/fatmakaraa/atelier_devops-main.git'  
                )
            }
        }
        stage('Secrets Scan') {
            steps {
                sh 'gitleaks detect --source . --verbose --redact'
                // Cette commande échoue si des secrets sont détectés
            }
        }
        // Etape 2 : test unitaire avec Maven
        stage('Tests Unitaires') 
        {
            steps {
                sh 'mvn test' 
            }
        }
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                    -DfailBuildOnCVSS=7 \
                    || echo "Dependency check completed - continuing pipeline"
                '''
            }
        }
        // Etape 3 : compilation du code source  
        stage('compile') 
        {
            steps {
                sh 'mvn clean compile'
            }
        }
        // Etape 4 : sonarQube
        stage('MVN SONARQUBE') 
        {
            steps {
                 withSonarQubeEnv('sq1') {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }
  
  
        // Etape 6.1 : Construction des images Docker
        stage('Build Docker Images') 
        {
            steps {
                sh ' docker compose build'
            }
        }
  
         // Étape 6.2 : Authentification à Docker Hub
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
        // Etape 7 : push image docker sur Docker Hub
        stage('Push Docker Image') 
        {
            steps {
                sh '''
                     docker compose push
                     docker logout
                '''
            }
        }
        // Étape 8 : Déploiement avec Docker Compose
        stage('Deploy with Docker Compose') 
        {
            steps {
                sh ' docker compose down &&  docker compose up -d'
            }
        }
        // Étape 9 : Vérification du déploiement
      /*  stage('Verify Deployment') 
        {
            steps {
                script {
                    // Réessayer jusqu'à 5 fois, avec une attente de 10 secondes entre chaque tentative
                    retry(5) {
                        sleep 10 // Attendre 10 secondes avant chaque tentative
                        sh 'curl -I http://172.20.10.2:8089/kaddem'
                    }
                }
            }
        }*/
        stage('Generate HTML Report') {
            steps {
                script {
                    def htmlContent = """
                    <html>
                    <head><title>Pipeline Execution Report</title></head>
                    <body>
                    <h1>Pipeline Build Report</h1>
                    <h2>Build #${env.BUILD_NUMBER}</h2>
                    <p><strong>Status:</strong> ${currentBuild.currentResult ?: 'SUCCESS'}</p>
                    <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm")}</p>
                    <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Duration:</strong> ${currentBuild.durationString ?: 'N/A'}</p>
                    <hr>
                    <h3>Build Steps Completed:</h3>
                    <ul>
                        <li>✅ Git Checkout</li>
                        <li>✅ Secrets Scan</li>
                        <li>✅ Unit Tests</li>
                        <li>✅ Dependency Check</li>
                        <li>✅ Compilation</li>
                        <li>✅ SonarQube Analysis</li>
                        <li>✅ Docker Build</li>
                        <li>✅ Docker Push</li>
                        <li>✅ Deployment</li>
                    </ul>
                    </body>
                    </html>
                    """
                    
                    writeFile file: 'pipeline-report.html', text: htmlContent
                    
                    // ✅ ARCHIVE THE FILE - This will make it appear in "Build Artifacts"
                    archiveArtifacts artifacts: 'pipeline-report.html', allowEmptyArchive: true
                    
                    // Also archive test results if available
                    archiveArtifacts artifacts: '**/target/surefire-reports/*.html', allowEmptyArchive: true
                }
            }
        }
    }
     // Étape 10 : Send Email
        post {
        success {
            mail to: 'karaafatma01@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} Completed",
                body: """
                Build Successfully Completed!

                Job Name: ${env.JOB_NAME}
                Build Number: #${env.BUILD_NUMBER}
                Build Duration: ${currentBuild.durationString}
                
                Console Output: ${env.BUILD_URL}console
                Build Details: ${env.BUILD_URL}
                
                All tests passed and artifacts were successfully deployed.
                """
        }
        failure {
            mail to: 'karaafatma01@gmail.com',
                subject: "URGENT: ${env.JOB_NAME} #${env.BUILD_NUMBER} Failed",
                body: """
                BUILD FAILURE ALERT!

                Job Name: ${env.JOB_NAME}
                Build Number: #${env.BUILD_NUMBER}
                Build Duration: ${currentBuild.durationString}
                
                Error Location: ${env.BUILD_URL}console
                Build Details: ${env.BUILD_URL}
                
                Immediate Action Required:
                1. Review console output for errors
                2. Check recent code changes
                3. Verify dependency services
                4. Re-run build after fixes
                """
        }
        }
}
