---
title: Example 2
date: 20220223
author: Adrián Martín García
---
# Example 2
```groovy
// Name of the repository and its download link
def map1 = [
  "repo1_map1"      : "<git-url>",
  "repo2_map1"      : "<git-url>"
]

def map2 = [
  "repo1_map2"      : "<git-url>",
  "repo2_map2"      : "<git-url>",
  "repo3_map2"      : "<git-url>"
]

def map3 = [
  "repo1_map3"      : "<git-url>",
  "repo2_map3"      : "<git-url>",
  "repo3_map3"      : "<git-url>"
]

// Environment where we want to deploy
ENVIRONMENTS = [
  "DEV": [
    "KUBERNETES": "kubernetes-agent-dev",
  ],
  "PRE": [
    "KUBERNETES": "kubernetes-agent-pre",
  ],
  "PRO": [
    "KUBERNETES": "kubernetes-agent-pro",
  ]
]

pipeline {

parameters {
  choice(name: 'environment', choices: ENVIRONMENTS.keySet() as List, description: 'List of all available environments')
}

agent {
  kubernetes {
    cloud     "kubernetes"
    label     "app-all-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    yamlFile  "pod.yaml"
    defaultContainer "alpine"
  }
}

stages {

stage('Preparing workspace') {
  steps {
    sh '''
      apk update
      apk add coreutils git openssh-client bash
    '''
  } // `step` closing
} // `stage` closing


stage (' Deploy phase 1') {
  steps {
    script {
      // Dynamic Stages are generated, as many compo repositories have been declared
      map1.each { entry ->
        stage (entry.key) {
          withCredentials([
          sshUserPrivateKey( credentialsId: 'credential-id', keyFileVariable: 'key') ]) {
            sh """
              mkdir -p $WORKSPACE/$entry.key
              export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${key}'
              git clone $entry.value $WORKSPACE/$entry.key
            """
          }
          // The last tag of each repository is obtained to launch its construction
          script{
            GIT_LATEST_TAG = sh (
              script: "cd $WORKSPACE/$entry.key && git tag | sort --version-sort | grep -i v | tail -1",
            returnStdout: true
            ).trim()
              print("debug(GIT_LATEST_TAG): " + GIT_LATEST_TAG)
          }
          // The job is built with the last tag found and the one with the indicated input parameter.
          build job: "Devops/job/$entry.key/${GIT_LATEST_TAG}",
            parameters: [
              string(name: 'environment',    value: params.environment  )
            ]
        }
      }
    }
  } // `step` closing
} // `stage` closing

stage (' Deploy phase 2') {
  steps {
    script {
      // Dynamic Stages are generated, as many compo repositories have been declared
      map2.each { entry ->
        stage (entry.key) {
          withCredentials([
          sshUserPrivateKey( credentialsId: 'credential-id', keyFileVariable: 'key') ]) {
            sh """
              mkdir -p $WORKSPACE/$entry.key
              export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${key}'
              git clone $entry.value $WORKSPACE/$entry.key
            """
          }
          // The last tag of each repository is obtained to launch its construction
          script{
            GIT_LATEST_TAG = sh (
              script: "cd $WORKSPACE/$entry.key && git tag | sort --version-sort | grep -i v | tail -1",
            returnStdout: true
            ).trim()
              print("debug(GIT_LATEST_TAG): " + GIT_LATEST_TAG)
          }
          // The job is built with the last tag found and the one with the indicated input parameter.
          build job: "Devops/job/$entry.key/${GIT_LATEST_TAG}",
            parameters: [
              string(name: 'environment',    value: params.environment  )
            ]
        }
      }
    }
  } // `step` closing
} // `stage` closing

stage (' Deploy phase 3') {
  steps {
    script {
      // Dynamic Stages are generated, as many compo repositories have been declared
      map3.each { entry ->
        stage (entry.key) {
          withCredentials([
          sshUserPrivateKey( credentialsId: 'credential-id', keyFileVariable: 'key') ]) {
            sh """
              mkdir -p $WORKSPACE/$entry.key
              export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${key}'
              git clone $entry.value $WORKSPACE/$entry.key
            """
          }
          // The last tag of each repository is obtained to launch its construction
          script{
            GIT_LATEST_TAG = sh (
              script: "cd $WORKSPACE/$entry.key && git tag | sort --version-sort | grep -i v | tail -1",
            returnStdout: true
            ).trim()
              print("debug(GIT_LATEST_TAG): " + GIT_LATEST_TAG)
          }
          // The job is built with the last tag found and the one with the indicated input parameter.
          build job: "Devops/job/$entry.key/${GIT_LATEST_TAG}",
            parameters: [
              string(name: 'environment',    value: params.environment  )
            ]
        }
      }
    }
  } // `step` closing
} // `stage` closing

} // all stages

} // pipeline
```