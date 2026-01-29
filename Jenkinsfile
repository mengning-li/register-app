pipeline {
  agent { label 'Jenkins-Agent' }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  tools {
    jdk 'Java17'
    maven 'Maven3'
  }

  environment {
    SONAR_TOKEN = credentials('jenkins-sonarqube-token')
  }

  stages {

    stage('Checkout') {
      steps {
        cleanWs()
        git branch: 'main',
            credentialsId: 'github',
            url: 'https://github.com/mengning-li/register-app.git'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -U clean package'
      }
    }

    stage('Test') {
      steps {
        sh 'mvn -B test'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
            sh 'mvn -B sonar:sonar'
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          def qg = waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
          echo "Quality Gate status: ${qg.status}"
          if (qg.status != 'OK') {
            error "Quality Gate failed: ${qg.status}"
          }
        }
      }
    }
  }

  post {
    always {
      junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
      archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, onlyIfSuccessful: false
    }
  }
}
