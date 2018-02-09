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
}

/* vim:set ft=groovy: */
