---
title: Example 3
date: 20220223
author: Adrián Martín García
---
# Example 3
```groovy

PROJECTS = [
  "": [],
  "project1": ["ENV_ZONE": "us-central1-b"],
  "project2": ["ENV_ZONE": "us-central1-b"]
]

// ALL ACTIONS
ACTIONS = ['', 'APPLY', 'DIFF', 'TEMPLATE']

// ALL RELEASES
RELEASES = [
  '',
  'APP1',
  'APP2',
  ]

pipeline {

parameters {
  choice(name: 'ACTION',   choices: ACTIONS,                   description: 'Action over tenant')
  choice(name: 'RELEASE',  choices: RELEASES,                  description: 'Release to deploy')
  choice(name: 'PROJECT',  choices: PROJECTS.keySet() as List, description: 'Deploy resource on account')
}

options {
  buildDiscarder(logRotator(numToKeepStr: '20'))
  ansiColor('xterm')
}

environment {
  env_CREDENTIALS_ID = "env-${params.PROJECT.toLowerCase()}"
  env_PROJECT        = "${params.PROJECT.toLowerCase()}"
  env_ZONE           = "${PROJECTS[params.PROJECT]["env_ZONE"].toLowerCase()}"
}

agent {
  kubernetes {
    cloud            "kubernetes"
    label            "provider-env-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    yamlFile         "pod.yaml"
    defaultContainer "provider-env"
  }
}

stages {

stage('(env) Prepare config') {
  when {
    allOf {
      expression { ACTION != '' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    withCredentials([file(credentialsId: env.env_CREDENTIALS_ID, variable: 'env_RESOURCE_PROJECT_FILE')]) {
      sh """#!/bin/bash
      # CONFIGURE RESOURCE PROJECT
      gcloud auth activate-service-account --key-file=${env_RESOURCE_PROJECT_FILE}

      # CONFIGURE KUBERNETES CREDENTIALS
      gcloud container clusters get-credentials ${env.env_PROJECT} --zone ${env.env_ZONE} --project ${env.env_PROJECT}
      """
    }
  } // `steps` closing
} // `stage (env) Prepare config` closing

stage('(Helmfile) Action') {
  when {
    allOf {
      expression { ACTION != '' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    script {
      // LOAD CREDENTIALS
      withCredentials([
        sshUserPrivateKey(credentialsId: "cred", keyFileVariable: 'id_rsa'),
        file(credentialsId: env.env_CREDENTIALS_ID, variable: 'env_RESOURCE_PROJECT_FILE')
      ]) {
          sh("""
          export GOOGLE_APPLICATION_CREDENTIALS=${env_RESOURCE_PROJECT_FILE}
          cd charts/
          helmfile deps
          helmfile -e ${env.env_PROJECT} -l name=${params.RELEASE.toLowerCase()} ${params.ACTION.toLowerCase()}
          """)
      }
    }
  } // `steps` closing
} // `stage (Helmfile) Action` closing

} // `all stages` closing

} // `pipeline` closing
```