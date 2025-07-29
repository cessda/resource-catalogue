pipeline {
  agent {
    label 'jnlp-himem'
  }

  environment {
    IMAGE_NAME = 'eosc-resource-catalogue'
    REGISTRY = 'europe-west1-docker.pkg.dev/cessda-prod/docker'
    DOCKER_IMAGE = ''
    DOCKER_TAG = ''
  }
  stages {
    stage('Validate Version & Determine Docker Tag') {
      steps {
        script {
          def POM_VERSION = sh(script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout | cut -c 1-5', returnStdout: true).trim()
          if (env.BRANCH_NAME == 'develop') {
            VERSION = POM_VERSION
            DOCKER_TAG = 'dev'
            echo "Detected development branch."
          } else if (env.BRANCH_NAME == 'master') {
            VERSION = POM_VERSION
            DOCKER_TAG = "${VERSION}-beta"
            echo "Detected release branch: ${VERSION}-beta"
          } else if (env.BRANCH_NAME.startsWith('release/')) {
            VERSION = env.BRANCH_NAME.split('/')[1]
            DOCKER_TAG = "${VERSION}-beta"
            echo "Detected release branch: ${VERSION}-beta"
          } else if (env.TAG_NAME != null) {
            VERSION = env.TAG_NAME.replaceFirst(/^v/, '')
            DOCKER_TAG = VERSION
            echo "Detected tag: ${env.TAG_NAME} (version ${VERSION})"
          } else {
            VERSION = POM_VERSION
            def branch = env.BRANCH_NAME.replace('/', '-')
            DOCKER_TAG = "${VERSION}-${branch}"
          }
          if ( POM_VERSION != VERSION ) {
            error("Version mismatch. \n\tProject's pom version:\t${POM_VERSION} \n\tBranch|Tag version:\t${VERSION}")
          }

          currentBuild.displayName = "${currentBuild.displayName}-${DOCKER_TAG}"
        }
      }
    }
    stage('Build Image') {
      steps{
        script {
          DOCKER_IMAGE = docker.build("${REGISTRY}/${IMAGE_NAME}:${DOCKER_TAG}", "--build-arg profile=beyond .")
        }
      }
    }
    stage('Upload Image') {
      when { // upload images only from the 'cessda' branch and tagged builds
        expression {
          return env.TAG_NAME != null || env.BRANCH_NAME == 'cessda'
        }
      }
      steps{
        script {
          sh """
              echo "Pushing image: ${DOCKER_IMAGE.id}"
              gcloud auth configure-docker ${ARTIFACT_REGISTRY_HOST}
          """
          DOCKER_IMAGE.push()
        }
      }
    }
    stage('Remove Image') {
      steps{
        script {
          sh "docker rmi ${DOCKER_IMAGE.id}"
        }
      }
    }
  }
  // post-build actions
  post {
    success {
      echo 'Build Successful'
    }
    failure {
      echo 'Build Failed'
    }
  }
}
