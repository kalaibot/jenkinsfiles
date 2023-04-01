def EXECUTOR_AGENT=null
def FILE_NAME=null
def MANIFEST_PATH=null

pipeline {
    
  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }
  
  environment {
    channel = '#devops-notifications'
  }
  
  agent none
  
  stages {
    stage('Choose Jenkins agent') {
      steps {
        script {
          if(params.environment == 'prod') {
            EXECUTOR_AGENT="prod-eks-agent"
            FILE_NAME="helm/environments/sa-east-1/cluster-2/airflow.yaml"
            MANIFEST_PATH="helm/manifests/sa-east-1/cluster-2/airflow/01_airflow.yaml"
          } else {
            EXECUTOR_AGENT="dev-eks-agent"
            FILE_NAME="helm/environments/sa-east-1/cluster-1/airflow.yaml"
            MANIFEST_PATH="helm/manifests/sa-east-1/cluster-1/airflow/01_airflow.yaml"
          }
        }
      }
    }
    stage('Checkout SCM') {
      agent { label EXECUTOR_AGENT }
      steps {
        git credentialsId: 'jenkins-github-token', url: 'https://github.com/kalaibot/airflow-helm.git'
      }
    }
    stage ('Setting airflow image tag in values.yaml') {
      agent { label EXECUTOR_AGENT }
      steps {
        script { def filename = FILE_NAME
        def data = readYaml file: filename
        // setting new image in the file
        data.airflow.image.tag = params.image
        data.airflow.config.AIRFLOW__KUBERNETES__WORKER_CONTAINER_TAG = params.image
        sh "rm $filename"
        writeYaml file: filename, data: data
        }
      }
    }
    stage('Creating manifests for the new image tag') {
      agent { label EXECUTOR_AGENT }
      steps {
        sh "helm template -f $FILE_NAME --name 'airflow' --namespace 'airflow' helm/charts/airflow-$airflow_chart_version/airflow airflow 2>/dev/null > $MANIFEST_PATH"
      }
    }
    stage('Deploying new airflow image tag') {
      agent { label EXECUTOR_AGENT }
      steps {
        sh "kubectl apply -f $MANIFEST_PATH"
      }
    }
  }
  post {
    always {
      node (EXECUTOR_AGENT) {
        sendEndBuildNotification(currentBuild.currentResult, env.channel, "Deployed to ${params.environment} sa-east-1 airflow -> ${params.image}")
        cleanWs()
      }
    }
  }
}
