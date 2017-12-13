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
            sh """
            for i in {1..10}
            do
                  do_something_here
                  echo $(date) > task1.log
            done
               """
        }
 }
 parallel Parallels: {
            stage ('Job Slave') {
            sh """
            for i in {1..10}
            do
                  do_something_here
                  echo $(date) > task1.log
            done
               """
            }
        }

sendMail(NICKNAME)
