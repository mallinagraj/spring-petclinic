pipeline {
    agent any
    
    options {
        skipDefaultCheckout()
    }

    
    tools {
        maven 'maven3'
        jdk 'jdk17'  // Spring PetClinic requires Java 17
    }
    
    environment {
        SONAR_URL = "http://3.110.136.214:9000"
        ARTIFACTORY_URL = "http://3.109.200.182:8081/artifactory"
        DOCKERHUB_USER = 'jayesh7744'
        IMAGE_NAME = 'spring-petclinic'
        IMAGE_TAG = "${BUILD_NUMBER}"
        MANAGER_EMAIL = 'mallinagraj2@gmail.com'  // UPDATE THIS
        DEVELOPER_EMAIL = 'devopsnagraj@gmail.com'  // UPDATE THIS
        SONAR_QUALITY_GATE_STATUS = 'PENDING'
        JUNIT_TEST_STATUS = 'PENDING'
        BUILD_STATUS = 'PENDING'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning GitHub Repo...'
                git branch: 'main', url: 'https://github.com/mallinagraj/spring-petclinic.git'
            }
        }
        
        stage('Build Maven Artifact') {
            steps {
                echo 'Building Spring PetClinic JAR package...'
                script {
                    try {
                        sh 'mvn clean package -DskipTests'
                        env.BUILD_STATUS = 'SUCCESS'
                        echo "✓ Build: SUCCESS"
                    } catch (Exception e) {
                        env.BUILD_STATUS = 'FAILED'
                        echo "✗ Build: FAILED"
                        currentBuild.result = 'FAILURE'
                        error("Build failed")
                    }
                }
            }
        }
        
        stage('Run JUnit Tests') {
            steps {
                echo 'Running JUnit Tests...'
                script {
                    try {
                        sh 'mvn test'
                        env.JUNIT_TEST_STATUS = 'PASSED'
                        echo "✓ JUnit Tests: PASSED"
                    } catch (Exception e) {
                        env.JUNIT_TEST_STATUS = 'FAILED'
                        echo "✗ JUnit Tests: FAILED"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Code Coverage Report') {
            steps {
                echo 'Generating JaCoCo Code Coverage Report...'
                sh 'mvn jacoco:report'
            }
            post {
                always {
                    jacoco(
                        execPattern: '**/target/jacoco.exec',
                        classPattern: '**/target/classes',
                        sourcePattern: '**/src/main/java',
                        exclusionPattern: '**/test/**'
                    )
                }
            }
        }
        
        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonarqube-token')
            }
            steps {
                script {
                    withSonarQubeEnv('SonarQubeServer') {
                        def scannerHome = tool name: 'SonarQubeScanner', 
                                              type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=spring-petclinic \
                                -Dsonar.projectName=spring-petclinic \
                                -Dsonar.sources=./src/main/java \
                                -Dsonar.tests=./src/test/java \
                                -Dsonar.java.binaries=./target/classes \
                                -Dsonar.host.url=${SONAR_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        
        stage('SonarQube Quality Gate') {
            steps {
                script {
                    echo '⏳ Waiting for SonarQube Quality Gate...'
                    timeout(time: 5, unit: 'MINUTES') {
                        try {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                env.SONAR_QUALITY_GATE_STATUS = "FAILED (${qg.status})"
                                echo "✗ SonarQube Quality Gate: FAILED"
                                currentBuild.result = 'UNSTABLE'
                            } else {
                                env.SONAR_QUALITY_GATE_STATUS = 'PASSED'
                                echo "✓ SonarQube Quality Gate: PASSED"
                            }
                        } catch (Exception e) {
                            env.SONAR_QUALITY_GATE_STATUS = 'TIMEOUT/ERROR'
                            echo "✗ SonarQube Quality Gate: TIMEOUT or ERROR"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        
        stage('Generate Test Report & Send to Manager') {
            steps {
                script {
                    // Get test statistics
                    def testResultAction = currentBuild.rawBuild.getAction(hudson.tasks.junit.TestResultAction.class)
                    def totalTests = testResultAction ? testResultAction.totalCount : 0
                    def failedTests = testResultAction ? testResultAction.failCount : 0
                    def passedTests = totalTests - failedTests
                    
                    def testResults = """
                    <html>
                    <head>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 0; padding: 0; background-color: #f5f5f5; }
                            .container { max-width: 800px; margin: 20px auto; background-color: white; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
                            .header { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 30px; text-align: center; }
                            .header h2 { margin: 0; font-size: 24px; }
                            .content { padding: 30px; }
                            .info-box { background-color: #f8f9fa; padding: 15px; border-radius: 5px; margin: 15px 0; }
                            .info-box strong { color: #333; }
                            .test-item { margin: 15px 0; padding: 20px; border-left: 5px solid #ddd; border-radius: 5px; background-color: #fafafa; }
                            .passed { border-left-color: #4CAF50; background-color: #f1f8f4; }
                            .failed { border-left-color: #f44336; background-color: #ffebee; }
                            .warning { border-left-color: #ff9800; background-color: #fff3e0; }
                            .test-stats { display: flex; justify-content: space-around; margin: 20px 0; }
                            .stat-box { text-align: center; padding: 15px; background-color: #f8f9fa; border-radius: 5px; flex: 1; margin: 0 10px; }
                            .stat-number { font-size: 32px; font-weight: bold; color: #667eea; }
                            .stat-label { color: #666; font-size: 14px; }
                            .recommendation { padding: 25px; margin: 25px 0; border-radius: 8px; text-align: center; font-size: 18px; font-weight: bold; }
                            .ready { background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%); color: white; }
                            .not-ready { background: linear-gradient(135deg, #eb3349 0%, #f45c43 100%); color: white; }
                            .links { background-color: #f8f9fa; padding: 20px; border-radius: 5px; margin: 20px 0; }
                            .links a { color: #667eea; text-decoration: none; font-weight: 500; }
                            .links a:hover { text-decoration: underline; }
                            .footer { background-color: #2c3e50; color: white; padding: 20px; text-align: center; }
                            ul { list-style-type: none; padding: 0; }
                            li { padding: 8px 0; }
                            li:before { content: "→ "; color: #667eea; font-weight: bold; }
                        </style>
                    </head>
                    <body>
                        <div class="container">
                            <div class="header">
                                <h2>🔔 Merge Request - Code Review Ready</h2>
                                <p style="margin: 10px 0 0 0; opacity: 0.9;">All automated tests have been completed</p>
                            </div>
                            
                            <div class="content">
                                <div class="info-box">
                                    <strong>📋 Project:</strong> Spring PetClinic (${env.JOB_NAME})<br>
                                    <strong>🔢 Build Number:</strong> #${env.BUILD_NUMBER}<br>
                                    <strong>🌿 Branch:</strong> main<br>
                                    <strong>📅 Build Date:</strong> ${new Date().format('yyyy-MM-dd HH:mm:ss')}<br>
                                    <strong>👤 Triggered By:</strong> ${env.DEVELOPER_EMAIL}
                                </div>
                                
                                <h3 style="color: #2c3e50; border-bottom: 2px solid #667eea; padding-bottom: 10px;">📊 Test Results Summary</h3>
                                
                                <div class="test-stats">
                                    <div class="stat-box">
                                        <div class="stat-number">${totalTests}</div>
                                        <div class="stat-label">Total Tests</div>
                                    </div>
                                    <div class="stat-box">
                                        <div class="stat-number" style="color: #4CAF50;">${passedTests}</div>
                                        <div class="stat-label">Passed</div>
                                    </div>
                                    <div class="stat-box">
                                        <div class="stat-number" style="color: #f44336;">${failedTests}</div>
                                        <div class="stat-label">Failed</div>
                                    </div>
                                </div>
                                
                                <div class="test-item ${env.BUILD_STATUS == 'SUCCESS' ? 'passed' : 'failed'}">
                                    <strong>1️⃣ Build Compilation:</strong> ${env.BUILD_STATUS}<br>
                                    <small style="color: #666;">Maven build and package creation</small>
                                </div>
                                
                                <div class="test-item ${env.JUNIT_TEST_STATUS == 'PASSED' ? 'passed' : 'failed'}">
                                    <strong>2️⃣ JUnit Unit Tests:</strong> ${env.JUNIT_TEST_STATUS}<br>
                                    <small style="color: #666;">${totalTests} tests executed | ${passedTests} passed | ${failedTests} failed</small>
                                </div>
                                
                                <div class="test-item ${env.SONAR_QUALITY_GATE_STATUS == 'PASSED' ? 'passed' : 'failed'}">
                                    <strong>3️⃣ SonarQube Quality Gate:</strong> ${env.SONAR_QUALITY_GATE_STATUS}<br>
                                    <small style="color: #666;">Code quality, security, and maintainability analysis</small>
                                </div>
                                
                                <div class="test-item passed">
                                    <strong>4️⃣ Code Coverage:</strong> GENERATED<br>
                                    <small style="color: #666;">JaCoCo coverage report available</small>
                                </div>
                                
                                <div class="recommendation ${env.BUILD_STATUS == 'SUCCESS' && env.JUNIT_TEST_STATUS == 'PASSED' && env.SONAR_QUALITY_GATE_STATUS == 'PASSED' ? 'ready' : 'not-ready'}">
                                    ${env.BUILD_STATUS == 'SUCCESS' && env.JUNIT_TEST_STATUS == 'PASSED' && env.SONAR_QUALITY_GATE_STATUS == 'PASSED' 
                                        ? '✅ ALL CHECKS PASSED<br><span style="font-size: 16px; font-weight: normal;">Code is ready to merge into main branch</span>' 
                                        : '⚠️ SOME CHECKS FAILED<br><span style="font-size: 16px; font-weight: normal;">Please review the failed checks before merging</span>'}
                                </div>
                                
                                <div class="links">
                                    <h3 style="color: #2c3e50; margin-top: 0;">🔗 Detailed Reports</h3>
                                    <ul>
                                        <li><a href="${env.BUILD_URL}">Jenkins Build Console</a></li>
                                        <li><a href="${env.BUILD_URL}testReport/">JUnit Test Report</a></li>
                                        <li><a href="${env.BUILD_URL}jacoco/">JaCoCo Code Coverage</a></li>
                                        <li><a href="${env.SONAR_URL}/dashboard?id=spring-petclinic">SonarQube Analysis Dashboard</a></li>
                                    </ul>
                                </div>
                                
                                <div style="background-color: #fff3cd; border-left: 4px solid #ffc107; padding: 15px; margin: 20px 0; border-radius: 5px;">
                                    <strong>⏰ Action Required:</strong> Please review the test results and provide your approval decision in Jenkins.
                                </div>
                            </div>
                            
                            <div class="footer">
                                <p style="margin: 0;">🤖 Automated CI/CD Pipeline Notification</p>
                                <p style="margin: 10px 0 0 0; opacity: 0.8; font-size: 12px;">Generated on ${new Date().format('yyyy-MM-dd HH:mm:ss')}</p>
                            </div>
                        </div>
                    </body>
                    </html>
                    """
                    
                    emailext(
                        subject: "🔔 [Spring PetClinic] Build #${env.BUILD_NUMBER} - Merge Request Ready ${env.BUILD_STATUS == 'SUCCESS' && env.JUNIT_TEST_STATUS == 'PASSED' && env.SONAR_QUALITY_GATE_STATUS == 'PASSED' ? '✅' : '⚠️'}",
                        body: testResults,
                        to: "${env.MANAGER_EMAIL}",
                        cc: "${env.DEVELOPER_EMAIL}",
                        mimeType: 'text/html',
                        attachLog: true
                    )
                    
                    echo "✓ Test report sent to manager: ${env.MANAGER_EMAIL}"
                }
            }
        }
        
        stage('Manager Approval for Deployment') {
            steps {
                script {
                    echo '⏳ Waiting for Manager Approval to proceed with deployment...'
                    
                    def allTestsPassed = (env.BUILD_STATUS == 'SUCCESS' && 
                                         env.JUNIT_TEST_STATUS == 'PASSED' && 
                                         env.SONAR_QUALITY_GATE_STATUS == 'PASSED')
                    
                    def approvalMessage = allTestsPassed 
                        ? """
✅ ALL TESTS PASSED!

Build Status: ${env.BUILD_STATUS}
JUnit Tests: ${env.JUNIT_TEST_STATUS}
SonarQube: ${env.SONAR_QUALITY_GATE_STATUS}

Ready to deploy to production?
""" 
                        : """
⚠️ SOME TESTS FAILED!

Build Status: ${env.BUILD_STATUS}
JUnit Tests: ${env.JUNIT_TEST_STATUS}
SonarQube: ${env.SONAR_QUALITY_GATE_STATUS}

Proceed with deployment anyway?
"""
                    
                    try {
                        timeout(time: 24, unit: 'HOURS') {
                            def userInput = input(
                                message: approvalMessage,
                                ok: 'Approve & Deploy',
                                submitter: 'admin,manager',  // UPDATE with actual Jenkins usernames
                                parameters: [
                                    choice(
                                        name: 'APPROVAL_DECISION',
                                        choices: ['Approve', 'Reject'],
                                        description: 'Do you approve this deployment?'
                                    ),
                                    text(
                                        name: 'APPROVAL_COMMENTS',
                                        defaultValue: '',
                                        description: 'Comments (optional)'
                                    )
                                ]
                            )
                            
                            if (userInput.APPROVAL_DECISION == 'Reject') {
                                error("Deployment rejected by manager")
                            }
                            
                            echo "✓ Deployment approved by manager"
                            echo "Comments: ${userInput.APPROVAL_COMMENTS}"
                        }
                    } catch (Exception e) {
                        echo "✗ Deployment rejected or timeout"
                        currentBuild.result = 'ABORTED'
                        error("Deployment was not approved: ${e.message}")
                    }
                }
            }
        }
        
        stage('Deploy JAR to Artifactory') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'jfrog-test', 
                        usernameVariable: 'JFROG_USER', 
                        passwordVariable: 'JFROG_PASSWORD')]) {
                        
                        sh '''
                            # Find the JAR file (exclude original)
                            JAR_FILE=$(find target -name '*.jar' -type f ! -name '*-original.jar' | head -n 1)
                            
                            if [ -z "$JAR_FILE" ]; then
                                echo "ERROR: No JAR file found in target directory"
                                exit 1
                            fi
                            
                            JAR_NAME=$(basename $JAR_FILE)
                            echo "Found JAR file: $JAR_NAME"
                            echo "Uploading to Artifactory..."
                            
                            # Upload JAR file directly to Artifactory using cURL
                            curl -v -u ${JFROG_USER}:${JFROG_PASSWORD} \
                                 -X PUT \
                                 "http://3.109.200.182:8081/artifactory/libs-release-local/${JAR_NAME}" \
                                 -T $JAR_FILE
                            
                            if [ $? -eq 0 ]; then
                                echo "✓ Successfully uploaded JAR to Artifactory"
                                echo "Artifact URL: http://3.109.200.182:8081/artifactory/libs-release-local/${JAR_NAME}"
                            else
                                echo "✗ Failed to upload JAR to Artifactory"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                echo "Building Docker image..."
                sh """
                    # Create Dockerfile if it doesn't exist
                    cat > Dockerfile <<'EOF'
FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
EOF
                    
                    # Build Docker image
                    docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                    
                    echo "✓ Docker image built successfully"
                """
            }
        }
        
        stage('Push Docker Image to DockerHub') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                script {
                    withCredentials([string(
                        credentialsId: 'dockerhub', 
                        variable: 'DOCKERHUB_TOKEN')]) {
                        sh """
                            echo "\$DOCKERHUB_TOKEN" | docker login -u ${DOCKERHUB_USER} --password-stdin
                            docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                            echo "✓ Successfully pushed Docker image to DockerHub"
                        """
                    }
                }
            }
        }
        
        stage('Send Deployment Success Notification') {
            when {
                expression { currentBuild.result != 'ABORTED' && currentBuild.result != 'FAILURE' }
            }
            steps {
                script {
                    def deploymentSuccess = """
                    <html>
                    <head>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 0; padding: 0; background-color: #f5f5f5; }
                            .container { max-width: 800px; margin: 20px auto; background-color: white; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
                            .header { background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%); color: white; padding: 30px; text-align: center; }
                            .content { padding: 30px; }
                            .success-icon { font-size: 60px; text-align: center; margin: 20px 0; }
                            .artifact-box { background-color: #f8f9fa; padding: 20px; border-radius: 8px; margin: 15px 0; border-left: 4px solid #38ef7d; }
                            .footer { background-color: #2c3e50; color: white; padding: 20px; text-align: center; }
                            .info-row { padding: 10px 0; border-bottom: 1px solid #eee; }
                            .label { font-weight: bold; color: #555; }
                            code { background-color: #f4f4f4; padding: 2px 6px; border-radius: 3px; font-family: monospace; }
                        </style>
                    </head>
                    <body>
                        <div class="container">
                            <div class="header">
                                <div class="success-icon">🎉</div>
                                <h2>Deployment Successful!</h2>
                                <p style="opacity: 0.9; margin: 10px 0 0 0;">Spring PetClinic has been deployed</p>
                            </div>
                            
                            <div class="content">
                                <div class="info-row">
                                    <span class="label">Project:</span> Spring PetClinic
                                </div>
                                <div class="info-row">
                                    <span class="label">Build Number:</span> #${env.BUILD_NUMBER}
                                </div>
                                <div class="info-row">
                                    <span class="label">Deployed At:</span> ${new Date().format('yyyy-MM-dd HH:mm:ss')}
                                </div>
                                <div class="info-row">
                                    <span class="label">Deployed By:</span> ${env.DEVELOPER_EMAIL}
                                </div>
                                
                                <h3 style="color: #2c3e50; margin-top: 30px;">📦 Deployed Artifacts</h3>
                                
                                <div class="artifact-box">
                                    <strong>✅ JAR Artifact</strong><br>
                                    <small>Uploaded to JFrog Artifactory</small><br>
                                    <code>libs-release-local/spring-petclinic-*.jar</code>
                                </div>
                                
                                <div class="artifact-box">
                                    <strong>✅ Docker Image</strong><br>
                                    <small>Pushed to DockerHub</small><br>
                                    <code>${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}</code><br>
                                    <code>${DOCKERHUB_USER}/${IMAGE_NAME}:latest</code>
                                </div>
                                
                                <h3 style="color: #2c3e50;">🚀 Deployment Commands</h3>
                                <div style="background-color: #f8f9fa; padding: 15px; border-radius: 5px; font-family: monospace; font-size: 13px;">
                                    # Pull and run the Docker image:<br>
                                    docker pull ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}<br>
                                    docker run -p 8080:8080 ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                                </div>
                                
                                <h3 style="color: #2c3e50;">🔗 Quick Links</h3>
                                <ul style="list-style: none; padding: 0;">
                                    <li style="padding: 5px 0;">→ <a href="${env.BUILD_URL}">Jenkins Build</a></li>
                                    <li style="padding: 5px 0;">→ <a href="http://3.109.200.182:8081/artifactory/libs-release-local/">JFrog Artifactory</a></li>
                                    <li style="padding: 5px 0;">→ <a href="https://hub.docker.com/r/${DOCKERHUB_USER}/${IMAGE_NAME}">DockerHub Repository</a></li>
                                </ul>
                            </div>
                            
                            <div class="footer">
                                <p style="margin: 0;">✨ Deployment completed successfully</p>
                                <p style="margin: 10px 0 0 0; opacity: 0.8; font-size: 12px;">Thank you for using our CI/CD pipeline</p>
                            </div>
                        </div>
                    </body>
                    </html>
                    """
                    
                    emailext(
                        subject: "✅ [Spring PetClinic] Build #${env.BUILD_NUMBER} - Deployment Successful 🎉",
                        body: deploymentSuccess,
                        to: "${env.DEVELOPER_EMAIL}",
                        cc: "${env.MANAGER_EMAIL}",
                        mimeType: 'text/html'
                    )
                    
                    echo "✓ Deployment success notification sent"
                }
            }
        }
    }
    
    post {
        success {
            echo "════════════════════════════════════════════════"
            echo "✓ Pipeline completed successfully!"
            echo "════════════════════════════════════════════════"
            echo "JAR artifact: Available in Artifactory"
            echo "Docker image: ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
            echo "Docker latest: ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
            echo "════════════════════════════════════════════════"
        }
        failure {
            script {
                def failureEmail = """
                <html>
                <body style="font-family: Arial, sans-serif;">
                    <div style="background: linear-gradient(135deg, #eb3349 0%, #f45c43 100%); color: white; padding: 30px; text-align: center;">
                        <h2>❌ Pipeline Failed</h2>
                    </div>
                    <div style="padding: 30px;">
                        <p><strong>Project:</strong> Spring PetClinic</p>
                        <p><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                        <p><strong>Status:</strong> FAILED</p>
                        <p><strong>Failed At:</strong> ${new Date().format('yyyy-MM-dd HH:mm:ss')}</p>
                        <hr>
                        <p style="color: #d32f2f;">⚠️ The pipeline encountered an error. Please check the console logs for details.</p>
                        <p><a href="${env.BUILD_URL}console" style="background-color: #d32f2f; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; display: inline-block;">View Console Output</a></p>
                    </div>
                </body>
                </html>
                """
                
                emailext(
                    subject: "❌ [Spring PetClinic] Build #${env.BUILD_NUMBER} - Pipeline Failed",
                    body: failureEmail,
                    to: "${env.DEVELOPER_EMAIL}",
                    cc: "${env.MANAGER_EMAIL}",
                    mimeType: 'text/html'
                )
            }
            echo "✗ Pipeline failed. Check the logs above."
        }
        aborted {
            script {
                def abortEmail = """
                <html>
                <body style="font-family: Arial, sans-serif;">
                    <div style="background-color: #ff9800; color: white; padding: 30px; text-align: center;">
                        <h2>⚠️ Deployment Rejected</h2>
                    </div>
                    <div style="padding: 30px;">
                        <p><strong>Project:</strong> Spring PetClinic</p>
                        <p><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                        <p><strong>Status:</strong> ABORTED</p>
                        <hr>
                        <p>The deployment was rejected by the manager or the approval timed out after 24 hours.</p>
                        <p>The build artifacts are available but were not deployed to production.</p>
                    </div>
                </body>
                </html>
                """
                
                emailext(
                    subject: "⚠️ [Spring PetClinic] Build #${env.BUILD_NUMBER} - Deployment Rejected",
                    body: abortEmail,
                    to: "${env.DEVELOPER_EMAIL}",
                    cc: "${env.MANAGER_EMAIL}",
                    mimeType: 'text/html'
                )
            }
            echo "⚠️ Pipeline aborted - Deployment not approve
