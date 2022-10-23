---
title: Example 6
date: 20220223
author: Adrián Martín García
---
# Example 6
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