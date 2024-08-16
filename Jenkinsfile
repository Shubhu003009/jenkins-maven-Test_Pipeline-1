pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    triggers {
        githubPush()
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
        // Artifact Retention Policies
        buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '2'))
    }

    stages {
        stage('Initialization') {
            steps {
                echo "=============== Initialization started ==============="
                script {
                    // Sending Slack notification at the start of the job
                    BUILD_TRIGGER_BY = currentBuild.getBuildCauses()[0].shortDescription + " / " + currentBuild.getBuildCauses()[0].userId
                    sendSlackMessage("#${env.BUILD_NUMBER} ${env.JOB_NAME} ${BUILD_TRIGGER_BY}")
                }
            }
        }
        stage("Test") {
            options {
                timeout(time: 4, unit: 'MINUTES') 
            }
            // Build Caching
            // Leverage Maven's build cache to avoid rebuilding unchanged components.
            steps {
                echo "=============== Test started ==============="
                sh '''
                mvn dependency:go-offline
                mvn test -Dmaven.repo.local=.m2/reposry
                '''
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
            options {
                timeout(time: 5, unit: 'MINUTES') 
            }
            // Optimize Build Artifacts
            // Compress build artifacts to save disk space and optimize storage.
            steps {
                echo "=============== Build started ==============="
                sh '''
                mvn package -Dmaven.repo.local=.m2/repository
                tar -czf target/my-artifact.tar.gz target/*.war
                '''
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
            input {
                message "deploy to production-server ?"
                ok "Yes"
            }
            options {
                timeout(time: 5, unit: 'MINUTES') 
            }
            steps {
                echo '=============== Deploy_Production started ==============='
                deploy adapters: [tomcat9(credentialsId: 'TOMCAT-DEPLOY_TEST', path: '', url: 'http://65.0.25.153:8081/')], contextPath: '/app', onFailure: false, war: '**/*.war'
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
            echo '=============== Pipeline state Success ==============='
            archiveArtifacts artifacts: 'target/my-artifact.tar.gz', allowEmptyArchive: true
            sendSlackMessage("#${env.BUILD_NUMBER} ${env.JOB_NAME} - Success")
        }
        failure {
            echo '=============== Pipeline state Failed ==============='
            script {
                def failureMessage = "#${env.BUILD_NUMBER} ${env.JOB_NAME} - Failed\n"
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
    def timeZone = TimeZone.getTimeZone('Asia/Kolkata') // Timezone for IST
    def date = new Date()
    def dateFormat = new java.text.SimpleDateFormat('yyyy-MM-dd HH:mm:ss a') // Adjust format as needed
    dateFormat.setTimeZone(timeZone)
    def localTime = dateFormat.format(date)
    try {
        slackSend(channel: 'mernstack-devops',
                  message: "${message}\nIST: ${localTime}\nBuild-url: ${env.BUILD_URL}",
                  )
    } catch (Exception e) {
        echo "Slack notification failed: ${e.message}"
    }
}

