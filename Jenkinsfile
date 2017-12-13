#!/usr/bin/env groovy

def sendMail(NICKNAME) {

    emailext (
      to: 'dmitriy.kozharsky@gmail.com',
      subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - ' + "${env.BUILD_STATUS}",
      body: 'Project: $PROJECT_NAME<br/>Build # $BUILD_NUMBER<br/>Build status: ' + "${env.BUILD_STATUS}" + '<br/>View job results: $BUILD_URL<br/>Console output:<br/>${BUILD_URL}consoleFull'
    )
 }
 
 node('ec2Slave') {
     
     stage('Run Tests') {
            parallel {
                stage('Test On Windows') {
                    agent {
                        label "windows"
                    }
                    steps {
                        sh "echo 'message master' > task_master.log && sleep 10"
                    }
                    post {
                        always {
                            sh "echo 'first done'"
                        }
                    }
                }
                stage('Test On Linux') {
                    agent {
                        label "linux"
                    }
                    steps {
                        sh "echo 'message slave' > task_slave.log && sleep 10"
                    }
                    post {
                        always {
                            sh "echo 'second done'"
                        }
                    }
                }
            }
        }
 }

sendMail(NICKNAME)
