#!/usr/bin/env groovy

def sendMail(NICKNAME) {
  if(NICKNAME == 'GSE'){ // not PR check, send email to everyone
    emailext (
      to: 'solariat.active_dev@genesys.com jopdevops@flugel.it',
      subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - ' + "${env.BUILD_STATUS}",
      body: 'Project: $PROJECT_NAME<br/>Build # $BUILD_NUMBER<br/>Build status: ' + "${env.BUILD_STATUS}" + '<br/>View job results: $BUILD_URL<br/>Console output:<br/>${BUILD_URL}consoleFull'
    )
  }
  else{ // PR check, send email to PR creator
    emailext (
      recipientProviders: [[$class: 'CulpritsRecipientProvider']],
      subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - ' + "${env.BUILD_STATUS}",
      body: 'Project: $PROJECT_NAME<br/>Build # $BUILD_NUMBER<br/>Build status: ' + "${env.BUILD_STATUS}" + '<br/>View job results: $BUILD_URL<br/>Console output:<br/>${BUILD_URL}consoleFull'
    )
  }
}

def cleanupSpace() {
	    sh """
            echo "DELETE EXITED CONTAINERS"
            sudo docker rm -fv \$(sudo docker ps -qa --no-trunc --filter "status=exited") || true
            echo "DELETE TEST CONTAINERS"
            sudo docker ps -a  | grep -E "fb_tests_mongo|fb_tests_kafka|mongo_unit_test" \
            | grep -E "hours|weeks|months|years" | awk '{print \$1}' | sort | uniq | xargs sudo docker rm -fv || true
            echo "DELETE UNUSED VOLUMES"
            sudo docker volume ls -qf dangling=true || true
            sudo docker volume rm \$(sudo docker volume ls -qf dangling=true) || true
            echo "DELETE UNUSED IMAGES"
            sudo docker rmi \$(sudo docker images --filter "dangling=true" -q --no-trunc) || true
            sudo docker images | grep -E "tango|tango_tests" | grep -v "base" | grep -E "hours|days" | awk '{print \$3}' | sort | uniq | xargs sudo docker rmi -f || true
            sudo docker images | grep "facebook-connector" | grep -v "base" | grep "hours ago" | awk '{print \$3}' | sort | uniq | xargs sudo docker rmi -f || true
            echo "DELETE ALL IMAGES OLDER THAN WEEK"
            sudo docker images | grep -E "weeks|months|years" | awk '{print \$3}' | sort | uniq | xargs sudo docker rmi -f || true
            echo "DELETE OLD PACKAGES"
            sudo find /var/www/joppkgs -mtime +1 -type f -delete || true
            """
}

node('ec2Slave') {
  env.PATH = "${tool 'nodejs'}/bin:${env.PATH}"
  env.BUILD_STATUS = 'SUCCESS'
  withCredentials([[
      $class           : 'AmazonWebServicesCredentialsBinding',
      credentialsId    : 'jenkins-aws',
      accessKeyVariable: 'AWS_ACCESS_KEY_ID',
      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
      ]]) {
    try {
        stage ('Clean Checkout') {
            cleanupSpace()
            dir('package') {
                deleteDir()
            }
            checkout scm
            VERSION = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            BRANCH = sh(returnStdout: true, script: "git branch -r --contains ${VERSION}").trim().split('/')[-1]
            sh """
            chown -R 1000.1000 /var/jenkins_home
            sudo docker kill \$(docker ps -q) || true
            sudo docker rm \$(docker ps -a -q) || true
            """
        }
        stage ('Scan Deps') {
            sh """ 
            rm -rf dependency-check*
            /home/ec2-user/dependency-check/bin/dependency-check.sh --project "TangoDev" --scan "solariat_bottle/src/solariat_bottle" --format "XML" --disableNSP "true" --enableExperimental --disablePyDist "false" --disablePyPkg "false"
            """
            dependencyCheckPublisher canComputeNew: false, defaultEncoding: '', healthy: '0', pattern: '', unHealthy: '1000'
        }
        stage ('Execute Nosetests'){
            dir ('solariat_bottle/src/solariat_bottle/dialog_engine/data') {
                git url: 'git@github.com:solariat/de-gaap-payloads.git', branch: 'master', credentialsId: 'jenkins-key'
            }
            sh """
            sudo \$(aws ecr get-login --no-include-email --region eu-west-2)
            sudo docker run -d --name mongo_unit_test_${VERSION} mongo:3.0 --storageEngine=wiredTiger
            sudo docker run -i --rm -v \$(pwd):/nosetests -e LOG_LEVEL=WARN --link mongo_unit_test_${VERSION}:mongo --name nosetests_${VERSION} ${ECR_REPO}/nosetests:latest
            sudo docker rm -fv mongo_unit_test_${VERSION}
            """
        }
        parallel buildIP: {
            stage ('Build IP') {
                env.WORKSPACE = pwd()
                env.HOME = pwd() + "/package"
                dir('deployments'){
                    git url: 'git@github.com:solariat/deployments.git', branch: 'master', credentialsId: 'jenkins-key'
                    sh "chown -R 1000.1000 /var/jenkins_home"
                    sh "bash build_wheels.sh"
                }
                sh """
                bash scripts/build_jop.sh ${NICKNAME} ${VERSION}
                chown -R 1000.1000 /var/jenkins_home
                cp ${HOME}/joppkg/*.tgz /var/www/joppkgs/
                """
            }
        }, buildTestsImage: {
            stage ('Build Protractor'){
                dir('solariat_bottle/src/solariat_bottle'){
                    sh "sudo docker build -t tango_tests:${VERSION} ."
                }
            }
        }, buildTangoImage: {
            stage ('Build Tango') {
            sh """
            bash scripts/build_docker.sh ${NICKNAME} ${VERSION} ${BRANCH} ${ECR_REPO}/tango
            """
            }
        }
        stage ('Get Mongo dump'){
            withAWS(credentials:'dump_mongo', region: 'eu-west-1') {
                s3Download(file:'dump.tar.gz', bucket:'genesys-deploy-dump', path:'dump.tar.gz', force:true)
            }
        }
        stage ('Deploy Containers') {
            sh """
            chown -R 1000.1000 /var/jenkins_home
            sudo \$(aws ecr get-login --no-include-email --region eu-west-2)
            sudo bash scripts/deploy_docker.sh ${VERSION}
            """
        }
        stage ('JS-Unit tests'){
            sh "sudo docker run -i --rm --net=host --privileged -v /dev/shm:/dev/shm -v \$(pwd):/mnt --name js_unit_tests tango_tests:${VERSION} js-unit-test"
            sh "chown -R 1000.1000 /var/jenkins_home"
            junit 'karma-results.xml'
            publishHTML (target: [
                  allowMissing: false,
                  alwaysLinkToLastBuild: false,
                  keepAll: false,
                  reportDir: '__coverage/PhantomJS 2.1.1 (Linux 0.0.0)/',
                  reportFiles: 'index.html',
                  reportName: "HTML Report",
                  reportTitles: ''
            ])
        }
        stage ('E2E tests'){
            sh """
            rm -rf allure-results 
            mkdir allure-results
            chown -R 1000.1000 /var/jenkins_home
            if [ ${NICKNAME} = "GSE" ]; then
                TESTSET="e2e-full"
            else
                TESTSET="e2e"
            fi
            sudo docker run -i --rm --net=host --privileged -v /dev/shm:/dev/shm -v \$(pwd)/allure-results:/tests/allure-results --name e2e_test tango_tests:${VERSION} \$TESTSET
            """
        }
        stage ('Push Tango'){
            sh """
            sudo \$(aws ecr get-login --no-include-email --region eu-west-2)
            sudo docker push ${ECR_REPO}/tango:${VERSION}
            """
        }
        stage ('Trigger E2E Monitors'){
            build job: 'E2E_Monitors',
            parameters: [
                [$class: 'StringParameterValue', name: 'VERSION', value: VERSION],
                [$class: 'StringParameterValue', name: 'BRANCH', value: BRANCH],
                [$class: 'StringParameterValue', name: 'NICKNAME', value: NICKNAME]
            ]
        }
        stage ('Push Tango latest'){
            if(NICKNAME == 'GSE' || NICKNAME == 'PRR'){
                sh """
                sudo \$(aws ecr get-login --no-include-email --region eu-west-2)
                sudo docker tag ${ECR_REPO}/tango:${VERSION} ${ECR_REPO}/tango:${BRANCH}
                sudo docker push ${ECR_REPO}/tango:${BRANCH}
                if [ ${BRANCH} = "dev" ]; then
                  sudo docker pull ${ECR_REPO}/tango:latest
                  sudo docker tag ${ECR_REPO}/tango:latest ${ECR_REPO}/tango:old_latest
                  sudo docker push ${ECR_REPO}/tango:old_latest
                  sudo docker rmi -f ${ECR_REPO}/tango:old_latest
                  sudo docker tag ${ECR_REPO}/tango:${VERSION} ${ECR_REPO}/tango:latest
                  sudo docker push ${ECR_REPO}/tango:latest
                fi
                """
            }
        }
    } catch(err) {
        env.BUILD_STATUS = 'FAILED'
        throw(err)
    } finally {
        stage ('Allure results'){
            allure results: [[path: 'allure-results']]
        }
        stage ('Post build'){
           if(NICKNAME == 'GSE_PR'){
            	sh """
		echo "PR check - deleting image in ECR with tag ${VERSION}"
                sudo \$(aws ecr get-login --no-include-email --region eu-west-2)
                sudo aws ecr batch-delete-image --repository-name tango --image-ids imageTag=${VERSION} --region eu-west-2            
		"""
            }
            cleanupSpace()
            //junit '**/nosetests.xml'
            step([$class: 'CoberturaPublisher', failNoReports: true, autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/coverage.xml', failUnhealthy: true, failUnstable: true, maxNumberOfBuilds: 0, onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false])
            if (env.BUILD_STATUS == 'FAILED'){
                // Build is FAILED
                sendMail(NICKNAME)
            } else if (currentBuild.getPreviousBuild()?.getResult().toString() == 'FAILURE') { 
                // Build is SUCCESSFUL
                sendMail(NICKNAME)
            }
        }
      }
    }
}
