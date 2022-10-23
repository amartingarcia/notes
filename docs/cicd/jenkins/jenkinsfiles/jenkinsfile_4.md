---
title: Example 4
date: 20220223
author: Adrián Martín García
---
# Example 4
```groovy
pipeline {

/////////////////////////////////////////////////////////////////////////////////////
// - Start pipeline                                                                //
/////////////////////////////////////////////////////////////////////////////////////

agent {
  kubernetes {
    label "image-updater"
    yamlFile "pod.yaml"
    defaultContainer "kaniko"
  }
}
options {
  disableConcurrentBuilds()
  buildDiscarder(logRotator(numToKeepStr: '10'))
}
triggers {
  // Every monday between 9:00 and 11:59
  cron('H(0-59) H(9-11) * * 1')
}

stages {

/////////////////////////////////////////////////////////////////////////////////////
// - Start stages                                                                  //
/////////////////////////////////////////////////////////////////////////////////////

stage('Preparing workspace') {
  environment {
    version = VersionNumber(versionNumberString: '${BUILD_YEAR}.${BUILD_MONTH}.${BUILDS_THIS_MONTH}')
  }
  steps {
    withCredentials([
      file(credentialsId: 'cred', variable: 'DOCKER_CONFIG'),
    ]) {
      sh 'cp "${DOCKER_CONFIG}" /kaniko/.docker/config.json'
    }
  }
}

// If you run a pull request for testing purposes will not create lastest tag
stage('Creating testing images') {
  when {
    branch "PR-*"
  }
  environment {
    version = VersionNumber(versionNumberString: '${BUILD_YEAR}.${BUILD_MONTH}.${BUILDS_THIS_MONTH}')
  }
  steps {
    sh '''#!/busybox/sh
      for IMAGE in $WORKSPACE/images/*; do
        IMAGE=$(basename $IMAGE)

        /kaniko/executor \
          --reproducible \
          --context="${WORKSPACE}/images/${IMAGE}" \
          --destination="artifact.com/devops//${IMAGE}:${version}"
      done
    '''
  }
}

stage('Creating new images') {
  when {
    branch "main"
  }
  environment {
    version = VersionNumber(versionNumberString: '${BUILD_YEAR}.${BUILD_MONTH}.${BUILDS_THIS_MONTH}')
  }
  steps {
    sh '''#!/busybox/sh
      for IMAGE in $WORKSPACE/images/*; do
        IMAGE=$(basename $IMAGE)
        
        /kaniko/executor \
          --reproducible \
          --context="${WORKSPACE}/images/${IMAGE}" \
          --destination="artifact.com/devops/${IMAGE}:${version}" \
          --destination="artifact.com//devops/${IMAGE}:latest"
      done
    '''
  }
}

} } // `steps` and `pipeline` closing
```