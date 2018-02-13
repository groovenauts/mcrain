pipeline {
  agent any
  stages {
    stage("test") {
      agent { label "unittest" }
      environment {
        PARALLEL_TEST_PROCESSORS = "1"
        BUNDLE_DISABLE_EXEC_LOAD = "1"
        DOCKER_HOST = "tcp://127.0.0.1:2375"
      }
      steps {
        sh '''#!/bin/bash -l
              set -xe
              export -p
              gem env
              bundle check || bundle install --jobs=4 --path=vendor/bundle
              bundle exec mcrain pull all
              bundle exec rake parallel:spec
          '''
      }
    }
  }
  post {
    changed {
      slackNotify(currentBuild.currentResult)
    }
    aborted {
      slackNotify("ABORTED")
    }
  }
}


def slackNotify(result){
  // https://github.com/jenkinsci/slack-plugin/issues/327
  def durationString = currentBuild.durationString.replace(' and counting', '')
  def notificationMessage = {resultString ->
    "${env.JOB_NAME} - ${currentBuild.displayName} ${resultString} after ${durationString} (<${env.RUN_DISPLAY_URL}|Open>)"
  }
  switch(result){
    case "SUCCESS":
      slackSend color: 'good', message: notificationMessage("Back to normal")
      break
    case "FAILURE":
      slackSend color: 'danger', message: notificationMessage("Failure")
      break
    case "UNSTABLE":
      slackSend color: 'danger', message: notificationMessage("Unstable")
      break
    case "ABORTED":
      slackSend color: 'warning', message: notificationMessage("Aborted")
      break
    default:
      slackSend color: 'warning', message: notificationMessage("Unknown")
  }
}

/* vim:set ft=groovy: */
