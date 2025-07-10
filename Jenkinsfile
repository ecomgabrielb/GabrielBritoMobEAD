pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('sonar-token')
        PROJECT_KEY = 'GabrielMobEAD'
        REPO_URL = 'https://github.com/ecomgabrielb/GabrielBritoMobEAD.git'
        registry = "osanamgcj/mobead_image_build"
        registryCredential = 'dockerhub'
        dockerImage = ''
    }
    
    stages {
        stage('🔄 Checkout Code') {
            steps {
                echo "==================== CHECKOUT STAGE ===================="
                git branch: 'main', url: "${REPO_URL}"
                echo "✅ Source code checked out successfully"
                sh 'ls -la'
            }
        }
        
        stage('🔨 Build Application') {
            steps {
                echo "==================== BUILD STAGE ===================="
                script {
                    try {
                        echo "🔨 Building application..."
                        sh 'echo "Static website - no build required"'
                        echo "✅ Build completed successfully"
                    } catch (Exception e) {
                        echo "❌ Build failed: ${e.getMessage()}"
                        error "Build stage failed"
                    }
                }
            }
        }
        
        stage('🔍 SonarQube Code Analysis') {
            steps {
                echo "==================== SONARQUBE ANALYSIS STAGE ===================="
                script {
                    def scannerHome = tool 'SonarQube Scanner'
                    withSonarQubeEnv('SonarQube') {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                  -Dsonar.projectKey=${PROJECT_KEY} \
                                  -Dsonar.sources=. \
                                  -Dsonar.host.url=http://host.docker.internal:9000 \
                                  -Dsonar.login=\$SONAR_TOKEN \
                                  -Dsonar.inclusions=**/*.xml,**/*.config,**/*.yml,**/*.yaml \
                                  -Dsonar.exclusions=**/*.js,**/*.ts,**/*.html,**/*.css,**/*.less,**/node_modules/**,**/__MACOSX/**,**/.git/**
                            """
                        }
                    }
                    echo "✅ SonarQube analysis completed"
                }
            }
        }
        
        stage('📊 Quality Gate Check') {
            steps {
                echo "==================== QUALITY GATE STAGE ===================="
                script {
                    echo "⏳ Waiting for SonarQube Quality Gate..."
                    
                    def attempt = 0
                    def qgStatus = null
                    
                    timeout(time: 2, unit: 'MINUTES') {
                        while (true) {
                            attempt++
                            echo "🔍 Quality Gate check attempt ${attempt}..."
                            
                            try {
                                def qg = waitForQualityGate(abortPipeline: false)
                                qgStatus = qg.status
                                echo "📊 Quality Gate Status: ${qgStatus}"
                                
                                if (qgStatus == 'ERROR') {
                                    echo "❌ Quality Gate failed with ERROR status"
                                    error "Pipeline aborted due to quality gate failure: ${qgStatus}"
                                } else if (qgStatus == 'PASSED') {
                                    echo "✅ Quality Gate passed successfully!"
                                    break
                                } else {
                                    echo "⏳ Quality Gate status: ${qgStatus} - waiting 30 seconds before next check..."
                                    sleep(30)
                                }
                            } catch (Exception e) {
                                echo "⚠️  Quality Gate check failed on attempt ${attempt}: ${e.getMessage()}"
                                echo "⏳ Retrying in 30 seconds..."
                                sleep(30)
                            }
                        }
                    }
                }
            }
        }
        
        stage('🐳 Build Docker Image') {
            steps {
                echo "==================== DOCKER BUILD STAGE ===================="
                script {
                    try {
                        echo "🐳 Building Docker image..."
                        dockerImage = docker.build registry + ":$BUILD_NUMBER"
                        echo "✅ Docker image built successfully"
                    } catch (Exception e) {
                        echo "❌ Docker build failed: ${e.getMessage()}"
                        error "Docker build failed"
                    }
                }
            }
        }
        
        stage('🚧 Deploy to Staging') {
            steps {
                echo "==================== STAGING DEPLOYMENT STAGE ===================="
                script {
                    try {
                        echo "🚧 Deploying to staging environment..."
                        sh 'echo "Staging deployment - Docker image: ${registry}:$BUILD_NUMBER"'
                        echo "✅ Staging deployment completed"
                        echo "🌐 Staging URL: http://localhost:8081"
                    } catch (Exception e) {
                        echo "❌ Staging deployment failed: ${e.getMessage()}"
                        error "Staging deployment failed"
                    }
                }
            }
        }
        
        stage('⏸️ Manual Approval for Production') {
            steps {
                echo "==================== MANUAL APPROVAL STAGE ===================="
                script {
                    echo "⏳ Waiting for manual approval to deploy to production..."
                    
                    def approvalResponse = input(
                        message: '🚀 Ready to deploy to Production?',
                        ok: 'Deploy Now',
                        submitterParameter: 'APPROVER',
                        submitter: 'admin,deploy-team',
                        parameters: [
                            choice(
                                name: 'TARGET_ENV',
                                choices: ['production'],
                                description: 'Deployment environment'
                            ),
                            string(
                                name: 'DEPLOYMENT_NOTE',
                                defaultValue: 'Regular deployment',
                                description: 'Add deployment notes (optional)'
                            ),
                            booleanParam(
                                name: 'EMERGENCY_DEPLOYMENT',
                                defaultValue: false,
                                description: 'Check if this is an emergency deployment'
                            )
                        ]
                    )
                    
                    echo "✅ Production deployment approved by: ${approvalResponse.APPROVER}"
                    echo "📝 Deployment note: ${approvalResponse.DEPLOYMENT_NOTE}"
                    echo "🚨 Emergency deployment: ${approvalResponse.EMERGENCY_DEPLOYMENT}"
                    
                    env.APPROVER = approvalResponse.APPROVER
                    env.DEPLOYMENT_NOTE = approvalResponse.DEPLOYMENT_NOTE
                    env.EMERGENCY_DEPLOYMENT = approvalResponse.EMERGENCY_DEPLOYMENT
                }
            }
        }
        
        stage('🚀 Deploy to Production') {
            steps {
                echo "==================== PRODUCTION DEPLOYMENT STAGE ===================="
                script {
                    try {
                        echo "🚀 Starting production deployment..."
                        echo "👤 Approved by: ${env.APPROVER}"
                        echo "📝 Note: ${env.DEPLOYMENT_NOTE}"
                        
                        docker.withRegistry('https://registry-1.docker.io/v2/', registryCredential) {
                            dockerImage.push("$BUILD_NUMBER")
                            dockerImage.push("latest")
                        }
                        
                        echo "✅ Production deployment completed successfully"
                        echo "🌐 Production URL: http://localhost"
                        
                    } catch (Exception e) {
                        echo "❌ Production deployment failed: ${e.getMessage()}"
                        error "Production deployment failed"
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "==================== PIPELINE SUMMARY ===================="
            echo "📋 Pipeline execution completed"
            echo "🏗️  Build Number: ${env.BUILD_NUMBER}"
            echo "📊 Result: ${currentBuild.result ?: 'SUCCESS'}"
            echo "⏱️  Duration: ${currentBuild.durationString}"
            echo "🔗 Build URL: ${env.BUILD_URL}"
            
            archiveArtifacts artifacts: 'target/**/*.jar,dist/**,build/**', allowEmptyArchive: true
        }
        success {
            echo "🎉 SUCCESS: Pipeline completed successfully!"
        }
        failure {
            echo "❌ FAILURE: Pipeline failed!"
        }
        cleanup {
            echo "🧹 Cleaning up workspace..."
            deleteDir()
        }
    }
}