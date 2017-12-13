#!/usr/bin/env groovy

def sendMail(NICKNAME) {

    emailext (
      to: 'dmitriy.kozharsky@gmail.com',
      subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - ' + "${env.BUILD_STATUS}",
      body: 'Project: $PROJECT_NAME<br/>Build # $BUILD_NUMBER<br/>Build status: ' + "${env.BUILD_STATUS}" + '<br/>View job results: $BUILD_URL<br/>Console output:<br/>${BUILD_URL}consoleFull'
    )
 }
 
 node('ec2Slave') {
      stage ('Job Master') {
          sh "echo 'message master' > task1.log"
        }
 parallel Parallels: {
            stage ('Job Slave') {
                sh "echo 'message slave' > task1.log"
            }
        }
 }

sendMail(NICKNAME)
