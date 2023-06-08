---
title: Jenkins
date: 20220223
author: Adrián Martín García
---
# Jenkins
[TBD]
## Jenkinsfiles
On this section declare some Jenkinsfile documentated.

### Example 1
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

### Example 2
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

### Example 3
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

### Example 4
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

### Example 5
```groovy
// @Library("sonar@lts") _

// Available Environments
ENVIRONMENTS = [
  "dev": [
    "KUBERNETES": "agent-dev",
    "POD":        "pod.yaml",
  ],
  "pro": [
    "KUBERNETES": "agent-pro",
    "POD":        "pod.yaml",
  ]
]

// Available Microservices
MICROSERVICES = [
  "ALL",
  "MICRO1",
  "MICRO2"
]


pipeline {

/////////////////////////////////////////////////////////////////////////////////////
// - Start pipeline                                                                //
/////////////////////////////////////////////////////////////////////////////////////

agent {
  kubernetes {
    cloud "kubernetes"
    label "builder"
    yamlFile "pod.yaml"
    defaultContainer "node"
  }
}

options {
  disableConcurrentBuilds()
}

parameters {
  choice(name: 'ENVIRONMENT',  choices: ENVIRONMENTS.keySet() as List,  description: 'Environment where deploy' )
  choice(name: 'MICROSERVICE', choices: MICROSERVICES, description: 'Microservice deploy' )
}

environment {
  DOCKER_REGISTRY    = "registry.com"
  REPOSITORY_NAME    = "repo"
  HELM_RELEASE       = "release"
  TAG                = "${env.GIT_BRANCH.toLowerCase()}" // This will have main, tag or pr-id
  TENANT             = "${params.ENVIRONMENT.toLowerCase()}"
  MICROSERVICE       = "${params.MICROSERVICE.toLowerCase()}"
  DOCKER_URL         = "${DOCKER_REGISTRY}/${REPOSITORY_NAME}"
  NAMESPACE          = "default"
}

stages {

/////////////////////////////////////////////////////////////////////////////////////
// - Start stages                                                                  //
/////////////////////////////////////////////////////////////////////////////////////

stage("Prepare python environment") {
  when {
    branch 'PR-*'
  }
  steps {
    container("python") {
      sh '''
        apt-get update
        apt-get install --no-install-recommends --yes git openssh-client curl make postgresql-client
        pip install --upgrade pip
        pip install -r requirements-tests.txt
      '''
      // Restore dumps for unit tests
      sh 'psql -h localhost -p 5432 -U postgres < $WORKSPACE/tools/database/postgres/security_assets.sql'
      // Ingest data in Elastic ephemeral for unit tests
      sh 'python3 $WORKSPACE/tools/database/elastic/elastic.py'
    }
  } // `step` closing
} // `stage` closing

stage('Tag release') {
  when {
    branch "main"
  }
  steps {
    container("semantic-release") {
      script {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'cred', keyFileVariable: 'id_rsa')
        ]) {
          sh '''
            export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${id_rsa}'
            apk add curl sed git openssh
            semantic-release --plugins "@semantic-release/git"
          '''
        }
      }
    }
  } // `step` closing
} // `stage` closing

stage('Tests') {
  when {
    branch 'PR-*'
  }
  environment {
    LC_ALL = 'en_US.UTF-8'
    LANG   = 'en_US.UTF-8'
  }
  failFast false
  parallel {
    stage('Checkstyle test') {
      steps {
        container("python") {
          sh 'make lint'
        }
      } // `step` closing
    } // `stage` closing
  } // `pararell` closing
} // `main stage` closing

stage('Update Catalogue') {
  when {
    branch "main"
  }
  steps {
    container("python") {
      script {
      withCredentials([
        sshUserPrivateKey(credentialsId: 'cred', keyFileVariable: 'id_rsa')
      ])
        {
          sh '''
            apt-get update
            apt-get install --no-install-recommends --yes git openssh-client
            export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${id_rsa}'
            git checkout $GIT_BRANCH
            git remote add versioner $GIT_URL
            git fetch versioner
          '''
          // Execute catalogue script
          sh 'python3 $WORKSPACE/tools/catalogue/example.py'
          // Push changes to bitbucket
          sh '''
            export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${id_rsa}'
            AUTHOR=\$(git show --format="%ae" $GIT_COMMIT | head -1 | awk -F'@' '{ print $1 }')
            EMAIL=\$(git show --format="%ae" $GIT_COMMIT | head -1)
            git config user.name \\"$AUTHOR\\" && git config user.email \\"$EMAIL\\"
            git add .
            git diff-index --quiet HEAD || git commit -m "docs(catalogue): Update catalogue"
            git push origin $GIT_BRANCH
          '''
        }
      }
    }
  }
}

stage('Build image') {
  when {
    beforeAgent true
    anyOf{
      tag pattern: "^v\\d+.\\d+.\\d+(-[0-9a-zA-Z\\.]+)?\$", comparator: "REGEXP"
      branch 'PR-*'
    }
  }
  steps {
    sh 'echo "${TAG}" > $WORKSPACE/VERSION'

    container("kaniko") {
      withCredentials([
        file(credentialsId: 'cred', variable: 'DOCKER_CONFIG'),])
        {
          sh 'cp "${DOCKER_CONFIG}" /kaniko/.docker/config.json'
        }

      sh '''#!/busybox/sh
        /kaniko/executor \
          --reproducible \
          --context="${WORKSPACE}" \
          --destination="${DOCKER_URL}:${TAG}"
      '''
    }
  } // `step` closing
} // `stage` closing

stage('Deploy preview environment') {
  when {
    beforeAgent true
    branch 'PR-*'
  }
  agent {
    kubernetes {
      cloud ENVIRONMENTS["DEV"]["KUBERNETES"]
      label "preview"
      yamlFile ENVIRONMENTS["DEV"]["POD"]
    }
  }
  environment {
    ingress = "app-${TAG}.${NAMESPACE}-dns.internal"
    preview_values = "ephemeral.yaml"
  }
  steps {
    container("helmfile") {
      dir("charts") {
        sh '''
          echo "appVersion: ${TAG}"                                 >> releases/base/${HELM_RELEASE}/Chart.yaml
          sed -i "s/- host: TO_BE_FILLED_BY_CI/- host: ${ingress}/"    bases/${preview_values}
        '''

        sh '''
          helmfile -l name=${HELM_RELEASE}-secrets -e DEV-ephemeral apply
        '''

        sh '''
          helmfile -e DEV-ephemeral apply
        '''
      }
    }
  } // `step` closing
} // `stage` closing

stage('Deploy environment: All microservices') {
  when {
    beforeAgent true
    tag pattern: "^v\\d+.\\d+.\\d+(-[0-9a-zA-Z\\.]+)?\$", comparator: "REGEXP"
    allOf{
      expression { params.MICROSERVICE == 'ALL'}
      expression { params.ENVIRONMENT == 'DEV' || params.ENVIRONMENT == 'PRO' }
    }
  }
  agent {
    kubernetes {
      cloud ENVIRONMENTS[params.ENVIRONMENT]["KUBERNETES"]
      label "deployer-${params.ENVIRONMENT.toLowerCase()}"
      yamlFile ENVIRONMENTS[params.ENVIRONMENT]["POD"]
    }
  }
  steps {
    container("helmfile") {
      dir("charts") {
        sh '''
          echo "appVersion: ${TAG}" >> releases/base/${HELM_RELEASE}/Chart.yaml
        '''

        sh '''
          helmfile -l name=${HELM_RELEASE}-secrets -e ${TENANT} apply
        '''

        sh '''
          helmfile -e ${TENANT} apply
        '''
      }
    }
  } // `step` closing
} // `stage` closing

stage('Deploy to environment: Single microservices') {
  when {
    beforeAgent true
    tag pattern: "^v\\d+.\\d+.\\d+(-[0-9a-zA-Z\\.]+)?\$", comparator: "REGEXP"
    anyOf{
      expression { params.MICROSERVICE != 'ALL'}
    }
  }
  agent {
    kubernetes {
      cloud ENVIRONMENTS[params.ENVIRONMENT]["KUBERNETES"]
      label "PRO-deployer-${params.ENVIRONMENT.toLowerCase()}"
      yamlFile ENVIRONMENTS[params.ENVIRONMENT]["POD"]
    }
  }
  steps {
    container("helmfile") {
      dir("charts") {
        sh '''
          echo "appVersion: ${TAG}" >> releases/base/${HELM_RELEASE}/Chart.yaml
        '''

        sh '''
          helmfile -l name=${HELM_RELEASE}-secrets -e ${TENANT} apply
        '''

        sh '''
          helmfile -l name=${HELM_RELEASE}-${MICROSERVICE} -e ${TENANT} apply
        '''
      }
    }
  } // `step` closing
} // `stage` closing

} // `all stages` closing
} // `pipeline` closing
```

### Example 6
```groovy
TENANTS = [
  "": [],
  "dev": [
    "AWS_DEFAULT_REGION":  "us-east-1",
    "AWS_CREDENTIALS":     "aws-dev",
    "AWS_ASSUME_ROLE_ARN": "arn:aws:iam::000000000000:role/dev",
    "AWS_EXTERNAL_ID":     "000000000000",
    "SSH_KEY":             "dev-key",
    "TENANT":              "dev",
  ],
  "pro": [
    "AWS_DEFAULT_REGION":  "us-east-1",
    "AWS_CREDENTIALS":     "aws-pro",
    "AWS_ASSUME_ROLE_ARN": "arn:aws:iam::111111111111:role/dev",
    "AWS_EXTERNAL_ID":     "111111111111",
    "SSH_KEY":             "pro-key",
    "TENANT":              "pro",
  ]
]

// @function: terraform
def terraform(status) {
  sh """#!/bin/bash
    # VARS
    export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
    export GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ${id_rsa}'
    export TF_VAR_aws_assume_role_arn="${AWS_ASSUME_ROLE_ARN}"
    export TF_VAR_aws_external_id="${AWS_EXTERNAL_ID}"

    # WOKRKDIR
    cd ${WORKSPACE}/terraform/tenant/${TENANT}/

    # Show current dir
    echo "###########################################"
    for i in \$(ls -d */ | cut -d " " -f10); do
      echo "Terraform directory: \${i}"

    done
    echo "###########################################"

    for i in \$(ls -d */ | cut -d " " -f10); do
      cd ${WORKSPACE}/terraform/tenant/${TENANT}/\${i}/
      echo "The current workdir is: \${i}"

      # EXECUTION
      ## get modules and instance backend
      terraform init \
        -backend-config="role_arn=\${AWS_ASSUME_ROLE_ARN}" \
        -backend-config="external_id=\${AWS_EXTERNAL_ID}"

      ## validate syntax and show changes
      terraform validate
      tflint --init

      ## conditional to deploy/destroy or plan
      if [[ "${status}" != "plan" ]]; then
        terraform ${status} -auto-approve
      else
        terraform ${status}
      fi
    done
  """
}

pipeline {

parameters {
  choice(name: 'ACTION',  choices: ['', 'DEPLOY', 'PLAN', 'DESTROY'], description: 'Action over tenant')
  choice(name: 'TENANT',  choices: TENANTS.keySet() as List,          description: 'Deploy tenant')
}

options {
  buildDiscarder(logRotator(numToKeepStr: '20'))
  ansiColor('xterm')
}

environment {
  ANSIBLE_FORCE_COLOR = true
  AWS_ASSUME_ROLE_ARN = "${TENANTS[params.TENANT]["AWS_ASSUME_ROLE_ARN"]}"
  AWS_CREDENTIALS     = "${TENANTS[params.TENANT]["AWS_CREDENTIALS"]}"
  AWS_DEFAULT_REGION  = "${TENANTS[params.TENANT]["AWS_DEFAULT_REGION"]}"
  AWS_EXTERNAL_ID     = "${TENANTS[params.TENANT]["AWS_EXTERNAL_ID"]}"
  SSH_KEY             = "${TENANTS[params.TENANT]["SSH_KEY"]}"
  TENANT              = "${TENANTS[params.TENANT]["TENANT"]}"
  TFLINT_LOG          = "info"
}

agent {
  kubernetes {
    cloud            "kubernetes"
    label            "k8s-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    yamlFile         "pod.yaml"
    defaultContainer "alpine"
  }
}

stages {

stage('(AWS) Prepare config') {
  when {
    allOf {
      expression { ACTION != '' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    withAWS(credentials: AWS_CREDENTIALS) {
      sh """#!/bin/bash
        # HOTFIX: config permission
        chown root:root /root/.ssh/config
        chmod 644 /root/.ssh/config

        # AWS configs
        mkdir -p ~/.aws
        ## config
        echo "[profile default]" >> ~/.aws/config
        echo "output = json" >> ~/.aws/config
        echo "region = ${env.AWS_DEFAULT_REGION}" >> ~/.aws/config
        echo "role_arn = ${env.AWS_ASSUME_ROLE_ARN}" >> ~/.aws/config
        echo "source_profile = default" >> ~/.aws/config
        echo "external_id = ${env.AWS_EXTERNAL_ID}" >> ~/.aws/config

        ## credentials
        echo "[default]" >> ~/.aws/credentials
        echo "aws_access_key_id = ${env.AWS_ACCESS_KEY_ID}" >> ~/.aws/credentials
        echo "aws_secret_access_key = ${env.AWS_SECRET_ACCESS_KEY}" >> ~/.aws/credentials
      """
    }
  } // `steps` closing
} // `stage (AWS) Prepare config` closing

stage('(Terraform) Review changes') {
  when {
    allOf {
      expression { ACTION == 'PLAN' || ACTION == 'DEPLOY' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    script {
      // LOAD CREDENTIALS
      withAWS(credentials: AWS_CREDENTIALS) {
        withCredentials([
          sshUserPrivateKey(credentialsId: "cred", keyFileVariable: 'id_rsa')
        ])
        {
          // execute terraform plan
          terraform('plan')
        }
      }
    }
  } // `steps` closing
} // `stage (Terraform) Review changes` closing

stage('(Terraform) Deploy infraestructure') {
  when {
    allOf {
      expression { ACTION == 'DEPLOY' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    script {
      // LOAD CREDENTIALS
      withAWS(credentials: AWS_CREDENTIALS) {
        withCredentials([
          sshUserPrivateKey(credentialsId: "cred", keyFileVariable: 'id_rsa')
        ])
        {
          // execute terraform apply
          terraform('apply')
        }
      }
    }
  } // `steps` closing
} // `stage (Terraform) Deploy infraestructure` closing

stage('(Terraform) Destroy infraestructure') {
  when {
    allOf {
      expression { ACTION == 'DESTROY' }
      branch 'main'
    }
    beforeAgent true
  }
  steps {
    script {
      // LOAD CREDENTIALS
      withAWS(credentials: AWS_CREDENTIALS) {
        withCredentials([
          sshUserPrivateKey(credentialsId: "cred", keyFileVariable: 'id_rsa')
        ])
        {
          // execute terraform destroy
          terraform('destroy')
        }
      }
    }
  } // `steps` closing
} // `stage (Terraform) Destroy infraestructure` closing

} // `all stages` closing

} // `pipeline` closing
```