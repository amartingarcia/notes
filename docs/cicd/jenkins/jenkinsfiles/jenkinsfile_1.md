---
title: Example 1
date: 20220220
author: Adrián Martín García
---
# Example 1
```groovy
// Available environments in which it can be deployed
ENVIRONMENTS = [
  "DEV": [
    "KUBERNETES": "<kubernetes-agent>",
    "POD":        "pod.yaml",
    "LABEL":      "label"
  ],
  "PRO": [
    "KUBERNETES": "<kubernetes-agent>",
    "POD":        "pod.yaml",
    "LABEL":      "label"
  ],
]

pipeline {

/////////////////////////////////////////////////////////////////////////////////////
// - Start pipeline                                                                //
/////////////////////////////////////////////////////////////////////////////////////

options {
  // It allows to keep the pipeline, for 8 hours.
  timeout(time: 8, unit: 'HOURS')
  disableConcurrentBuilds()
}

parameters {
  // Gets a list of available environments to deploy, and offers them as input parameters
  choice(name: 'environment', choices: ENVIRONMENTS.keySet() as List, description: 'Environment where deploy' )
}

triggers { 
  // UTC - will take place between 23:01 and 23:09 UTC+2 ("H" allows you to choose from a range of time) 
  cron( env.BRANCH_NAME.equals('main')  ? 'H(1-29) 21 * * *' : '')
}

// By default everything runs in the cluster. Except the execution of the scripts.
agent {
  kubernetes {
    cloud "<kubernetes-agent>"
    label "label-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    yamlFile "pod.yaml"
    defaultContainer "alpine"
  }
}

stages {

/////////////////////////////////////////////////////////////////////////////////////
// - Start stages                                                                  //
/////////////////////////////////////////////////////////////////////////////////////

// Stage that allows the creation of tags through semantic-release
stage('Semantic Release') {
  when {
    // It will only be executed when it is built in any of the defined branches
    anyOf{
      branch "main"
      branch "dev"
    }
    beforeAgent true
  }
  steps {
    container('node'){
      withCredentials([
      sshUserPrivateKey(credentialsId: 'credential-id', keyFileVariable: 'credential-key-file')]) {
        sh """
          export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${credential-key-file}'
          apk add nodejs npm git openssh bash coreutils
          npm install -g semantic-release semantic-release/git -D
          semantic-release --debug --plugins "@semantic-release/git"
        """
      }
    }
  } // `step` closing
} // `stage` closing

// Stage that allows to obtain the last tag created in the repository, and to build it
stage('Calling tag job: PRO') {
  when {
    allOf{
      branch "main"
      triggeredBy 'TimerTrigger'
    }
    beforeAgent true
  }
  steps {
    container('alpine'){
      // Get the last tag created for main branch. Example, main: v1.1.1.
    script{
      sh 'apk add coreutils git'
      MAIN_GIT_LATEST_TAG = sh (
        script: "git tag | sort --version-sort | grep -vi 'dev' | tail -1",
        returnStdout: true
      ).trim()
        print("debug(MAIN_GIT_LATEST_TAG): " + MAIN_GIT_LATEST_TAG)
    }
    // It will run the job, with the last tag created for the main branch.
    build job: "devops/job/${MAIN_GIT_LATEST_TAG}", wait: false,
      parameters: [
        string(name: 'environment',    value: 'PRO'  )
      ]
    }
  } // `step` closing
} // `stage` closing

// Stage that allows to install all the necessary packages to execute the script
stage('Preparing workspace') {
  when {
    allOf{
      tag pattern: "^v\\d+.\\d+.\\d+(-[0-9a-zA-Z\\.]+)?\$", comparator: "REGEXP"
    }
  }
  steps {
    container('python'){
      // Prepare apk cache
    sh '''
      mkdir $WORKSPACE/apt-cache
      ln -sf /var/cache/apt $WORKSPACE/apt-cache
      apt update
    '''
    // Download Python dependencies while builinding virtualenv
    sh '''
      apt install -y virtualenv
    '''
    // Prepare virtualenv
    sh '''
      mkdir $WORKSPACE/virtualenv
      virtualenv -p python3 virtualenv/.venv
      $WORKSPACE/virtualenv/.venv/bin/pip3 install wheel
      $WORKSPACE/virtualenv/.venv/bin/pip3 install -r $WORKSPACE/requirements.txt
    '''
    // Stash created environment
    stash includes: 'apt-cache/**',  name: 'apt-cache'
    stash includes: 'virtualenv/**', name: 'virtualenv'
    }
  } // `step` closing
} // `stage` closing

// Stage that allows to execute the script
stage('Deploy environment') {
  agent {
    // Will obtain the data of the selected environment during the construction of the tag in Jenkins
    kubernetes {
      cloud ENVIRONMENTS[params.environment]["KUBERNETES"]
      label ENVIRONMENTS[params.environment]["LABEL"]
      yamlFile ENVIRONMENTS[params.environment]["POD"]
    }
  }
  when {
    allOf{
      // will only run if the regex matches
      tag pattern: "^v\\d+.\\d+.\\d+(-[0-9a-zA-Z\\.]+)?\$", comparator: "REGEXP"
    }
    beforeAgent true
  }
  steps {
    container('python'){
      // Unstash files from cluster
      unstash 'apt-cache'
      unstash 'virtualenv'
      retry(3){
        sh 'ln -sf $WORKSPACE/apt-cache/apt /var/cache/apt'
        sh 'apt update && apt install -y virtualenv'
        // Data source with the variables necessary for the execution of the script
        sh '''#!/bin/bash
          echo "Exec script"
        '''
      }
    }
  } // `step` closing
} // `stage` closing

} // `all stages` closing 

post {
  // If the execution of the pipeline failed or aborted, an email will be sent to the recipients.
  failure {
    mail body: "<b>Jenkins</b><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL build: ${env.BUILD_URL}",
    subject: "Jenkins Build Failed: Project name -> ${env.JOB_NAME}",
    mimeType: 'text/html',
    to: ""
  }
  aborted {
    mail body: "<b>Jenkins</b><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL build: ${env.BUILD_URL}",
    subject: "Jenkins Build Aborted: Project name -> ${env.JOB_NAME}",
    mimeType: 'text/html',
    to: ""
  }
} // `post actions` closing

} // `pipeline` closing
```