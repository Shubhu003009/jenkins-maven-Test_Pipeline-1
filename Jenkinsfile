pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        STAGE_FAILED = ''
        STAGE_ERROR = ''
    }
    // Documentation and Comments
    // This pipeline builds, tests, and deploys a Java application.
    // It includes stages for testing, building, and deploying to production.
    // Each stage includes a timeout to prevent long-running builds.
    // Artifacts are compressed and archived, and Slack notifications are sent on build success or failure.

    options {
        sendSlackMessage("Jenkins job ${env.JOB_NAME} (#${env.BUILD_NUMBER}) started by ${env.BUILD_USER} at ${new Date().format('HH:mm:ss')}\nBuild URL: ${env.BUILD_URL}")
        // Artifact Retention Policies
        buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '2'))
    }

    stages {
        stage("Test") {
            // Build Caching
            // Leverage Maven's build cache to avoid rebuilding unchanged components.
            options {
                timeout(time: 4, unit: 'MINUTES') 
            }
            steps {
                sh "mvn dependency:go-offline" // Pre-fetch dependencies to speed up the build process
                sh "mvn test -Dmaven.repo.local=.m2/repository"
            }
            post {
                failure {
                    script {
                        env.STAGE_FAILED = "Test"
                        env.STAGE_ERROR = currentBuild.description
                    }
                }
            }
        }
        stage("Build") {
            // Optimize Build Artifacts
            // Compress build artifacts to save disk space and optimize storage.
            options {
                timeout(time: 5, unit: 'MINUTES') 
            }
            steps {
                sh "mvn package -Dmaven.repo.local=.m2/repository"
                sh 'tar -czf target/my-artifact.tar.gz target/*.jar'
            }
            post {
                failure {
                    script {
                        env.STAGE_FAILED = "Build"
                        env.STAGE_ERROR = currentBuild.description
                    }
                }
            }
        }
        stage("Deploy_Production") {
            // This stage is for deploying the application to production
            when {
                branch 'main'
            }
            input {
                message "deploy to production-server ?"
                ok "Yes"
            }
            options {
                timeout(time: 5, unit: 'MINUTES') 
            }
            steps {
                deploy adapters: [tomcat9(credentialsId: 'TOMCAT-DEPLOY_TEST', path: '', url: 'http://65.0.25.153:8081/')], contextPath: '/app', onFailure: false, war: '**/*.war'
                echo 'Deploying to production'
            }
            post {
                failure {
                    script {
                        env.STAGE_FAILED = "Deploy_Production"
                        env.STAGE_ERROR = currentBuild.description
                    }
                }
            }
        }
    }
    post {
        success {
            echo "======== Pipeline state successful ========"
            archiveArtifacts artifacts: 'target/my-artifact.tar.gz', allowEmptyArchive: true
            sendSlackMessage("Pipeline ${env.JOB_NAME} (#${env.BUILD_NUMBER}) - Success")
        }
        failure {
            script {
                def failureMessage = "Pipeline ${env.JOB_NAME} (#${env.BUILD_NUMBER}) - Failed\n"
                if (env.STAGE_FAILED) {
                    failureMessage += "Stage Failed: ${env.STAGE_FAILED}\nError: ${env.STAGE_ERROR}"
                }
                sendSlackMessage(failureMessage)
            }
        }
    }
}

// Function to send a message to Slack
def sendSlackMessage(String message) {
    try {
        slackSend(channel: 'mernstack-devops',
                  message: "${message}\nBuild URL: ${env.BUILD_URL}",
                  )
    } catch (Exception e) {
        echo "Slack notification failed: ${e.message}"
    }
}