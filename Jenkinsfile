#!/usr/bin/env groovy

def sendMail(NICKNAME) {

    emailext (
      to: 'dmitriy.kozharsky@gmail.com',
      subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - ' + "${env.BUILD_STATUS}",
      body: 'Project: $PROJECT_NAME<br/>Build # $BUILD_NUMBER<br/>Build status: ' + "${env.BUILD_STATUS}" + '<br/>View job results: $BUILD_URL<br/>Console output:<br/>${BUILD_URL}consoleFull'
    )
 }
 
 node('ec2Slave') {
    

            parallel buildOne: {
            stage ('Build One') {
                    sh "echo 'one'"
                    sh "sleep 100"

            }
        }, buildTwo: {
            stage ('Build Two'){
                    sh "echo 'two'"
                    sh "sleep 100"
            }
            }, buildThree: {
            stage ('Build Three'){
                    sh "echo 'three'"
                    sh "sleep 100"

            }
        }
     stage ('Build Four'){
                    sh "echo 'four'"
                    sh "sleep 100"
     }

 }

sendMail(NICKNAME)
