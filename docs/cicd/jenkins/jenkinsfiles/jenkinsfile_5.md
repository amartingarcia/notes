---
title: Example 5
date: 20220223
author: Adrián Martín García
---
# Example 5
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