def DOCKER_IMAGE = null
def DOCKER_TAG = ''
def DOCKER_IMAGE_SHA = ''

pipeline {
  agent {
    label 'jnlp-himem'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds(abortPrevious: true)
    timeout(time: 30, unit: 'MINUTES')
    timestamps()
  }

  environment {
    IMAGE_NAME = 'eosc-resource-catalogue'
    REGISTRY = 'europe-west1-docker.pkg.dev/cessda-prod/docker'
    DOCKER_BUILDKIT = '1'
  }

  stages {

    stage('Determine Docker Tag') {
      steps {
        script {
          DOCKER_TAG = sh(script: "./mvnw help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
          echo "Docker tag: ${DOCKER_TAG}"
          currentBuild.displayName = "${currentBuild.displayName}-${DOCKER_TAG}"
        }
      }
    }

    stage('Test and Build Image') {
      parallel {

        stage('Test') {
          when { expression { return env.TAG_NAME == null } }
          steps {
            catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
              withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                sh './mvnw -B -T 1C verify -DfailBuildOnCVSS=11'
              }
            }
          }
          post {
            always {
              junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml, **/target/failsafe-reports/TEST-*.xml'
              recordCoverage(
                tools: [[parser: 'JACOCO', pattern: '**/target/site/jacoco/jacoco.xml, **/target/site/jacoco-it/jacoco.xml, **/target/site/jacoco-aggregate/jacoco.xml']],
                sourceDirectories: [
                  [path: 'resource-catalogue-api/src/main/java'],
                  [path: 'resource-catalogue-elastic/src/main/java'],
                  [path: 'resource-catalogue-jms/src/main/java'],
                  [path: 'resource-catalogue-model/src/main/java'],
                  [path: 'resource-catalogue-model-lot1/src/main/java'],
                  [path: 'resource-catalogue-rest/src/main/java'],
                  [path: 'resource-catalogue-service/src/main/java']
                ]
              )
            }
          }
        }

        stage('Build Image') {
          steps {
            script {
              DOCKER_IMAGE = docker.build("${REGISTRY}/${IMAGE_NAME}:${DOCKER_TAG}", "--build-arg profile=beyond --build-arg skipTests=true .")
              DOCKER_IMAGE_SHA = sh(script: "docker inspect --format='{{.Id}}' ${DOCKER_IMAGE.id}", returnStdout: true).trim()
            }
          }
        }

      }
    }

    stage('Upload Image') {
      when { // upload images only from the 'cessda' branch and tagged builds
        expression {
          return env.TAG_NAME != null || env.BRANCH_NAME == 'cessda'
        }
      }
      steps {
        script {
          sh """
            gcloud auth configure-docker ${ARTIFACT_REGISTRY_HOST}
          """
          DOCKER_IMAGE.push()
          if (DOCKER_TAG.endsWith('-SNAPSHOT')) {
            DOCKER_IMAGE.push("dev")
          } else {
            def minorTag = DOCKER_TAG.tokenize('.').take(2).join('.')
            DOCKER_IMAGE.push(minorTag)
            DOCKER_IMAGE.push("latest")
          }
        }
      }
    }
  }

  post {
    always {
      script {
        if (DOCKER_IMAGE_SHA) {
          sh "docker rmi -f ${DOCKER_IMAGE_SHA} || true"
        }
      }
    }
    success {
      echo 'Build Successful'
      build job: 'cessda.resource-catalogue.deploy/main', parameters: [string(name: 'BACKEND_IMAGE_TAG', value: DOCKER_TAG)], wait: false
    }
  }
}
