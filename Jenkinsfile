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
          sh "echo 'message master' > task_master.log && sleep 10"
        }

        parallel Parallels: {
                 phase1: { sh "sleep 10" },
                 phase2: { sh "sleep 10" }
            
        }
    }

 

sendMail(NICKNAME)
