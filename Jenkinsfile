@Library('dynatrace@master') _

def _VERSION = 'UNKNOWN'
def _TAG = 'UNKNOWN'
def _TAG_DEV = 'UNKNOWN'
def _TAG_STAGING = 'UNKNOWN'


pipeline {
  agent {
    label 'maven'
  }
  environment {
    APP_NAME = "carts"
    VERSION = readFile('version').trim()
    ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/sockshop-registry/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}:${env.VERSION}"
  }
  stages {
    stage('Maven build') {
      steps {
        checkout([$class: 'GitSCM', 
          branches: [[name: "${env.BRANCH_NAME}"]], 
          doGenerateSubmoduleConfigurations: false, 
          extensions: [], 
          userRemoteConfigs: [[url: "${env.REPOSITORY_NAME}"]]
        ])
        script {
          _VERSION = readFile('version').trim()
          _TAG = "${env.DOCKER_REGISTRY_URL}:5000/sockshop-registry/${env.ARTEFACT_ID}"
          _TAG_DEV = "${_TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
          _TAG_STAGING = "${_TAG}:${env.VERSION}"
        }
        container('maven') {
          sh "cd ${env.APP_NAME} && mvn -B clean package"
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "dcd ${env.APP_NAME} && ocker build -t ${_TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'registry-creds', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "cd ${env.APP_NAME} && docker login --username=anything --password=${TOKEN} ${env.DOCKER_REGISTRY_URL}:5000"
            sh "cd ${env.APP_NAME} && docker tag ${_TAG_DEV} ${_TAG_DEV}"
            sh "cd ${env.APP_NAME} && docker push ${_TAG_DEV}"
          }
        }
      }
    }
    
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "cd ${env.APP_NAME} && sed -i 's#image: .*#image: ${_TAG_DEV}#' manifest/carts.yml"
          sh "cd ${env.APP_NAME} && sed -i 's#value: to-be-replaced-by-jenkins.*#value: ${env.VERSION}-${env.BUILD_NUMBER}#' manifest/carts.yml"
          sh "cd ${env.APP_NAME} && kubectl -n dev apply -f manifest/carts.yml"
        }
      }
    }
    /*
    stage('DT Deploy Event') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
          createDynatraceDeploymentEvent(
          envId: 'Dynatrace Tenant',
          tagMatchRules: [
              [
              meTypes: [
                  [meType: 'SERVICE']
              ],
              tags: [
                  [context: 'CONTEXTLESS', key: 'app', value: "${env.APP_NAME}"],
                  [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
              ]
              ]
          ]) {
          }
      }
    }
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        sleep 150

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx', 
              resultsDir: "HealthCheck_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('jmeter') {
          script {
            def status = executeJMeter (
              scriptName: "jmeter/${env.APP_NAME}_load.jmx", 
              resultsDir: "FuncCheck_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "FuncCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Functional check in dev failed."
            }
          }
        }
      }
    }
    */
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          withCredentials([usernamePassword(credentialsId: 'registry-creds', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "cd ${env.APP_NAME} && docker login --username=anything --password=${TOKEN} ${env.DOCKER_REGISTRY_URL}:5000"
            sh "cd ${env.APP_NAME} && docker tag ${_TAG_DEV} ${_TAG_STAGING}"
            sh "cd ${env.APP_NAME} && docker push ${_TAG_STAGING}"
          }
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${_TAG_STAGING}"),
            string(name: 'VERSION', value: "${_VERSION}")
          ]
      }
    }
  }
}
