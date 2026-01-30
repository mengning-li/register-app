pipeline {
  agent { label 'Jenkins-Agent' }
  
  tools {
    jdk 'Java17'
    maven 'Maven3'
  }
  
  environment {
    APP_NAME = "register-app"
    RELEASE = "1.0.0"
    DOCKER_USER = "mengningli"
    DOCKER_PASS = 'dockerhub'
    IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
    IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
  }
  
  parameters {
    booleanParam(name: 'DOCKER_PUSH', defaultValue: false, description: 'Build and push Docker image?')
  }
  
  options {
    timestamps()
  }
  
  stages {
    stage('Cleanup Workspace') {
      steps {
        cleanWs()
      }
    }
    
    stage('Checkout from SCM') {
      steps {
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/mengning-li/register-app.git'
      }
    }
    
    stage('Build Application') {
      steps {
        sh 'mvn -B -U clean package'
      }
    }
    
    stage('Test Application') {
      steps {
        sh 'mvn -B test'
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
          archiveArtifacts artifacts: '**/target/*.jar, **/target/*.war', allowEmptyArchive: true, fingerprint: true
        }
      }
    }
    
    stage('SonarQube Analysis') {
      steps {
        script {
          withSonarQubeEnv('sonarqube-server') {
            sh 'mvn -B sonar:sonar'
          }
        }
      }
    }
    
    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    
    stage('Build Docker Image') {
      when { 
        expression { return params.DOCKER_PUSH } 
      }
      steps {
        script {
          docker.withRegistry('', DOCKER_PASS) {
            docker_image = docker.build "${IMAGE_NAME}:${IMAGE_TAG}"
          }
        }
      }
    }
    
    stage('Push Docker Image') {
      when { 
        expression { return params.DOCKER_PUSH } 
      }
      steps {
        script {
          docker.withRegistry('', DOCKER_PASS) {
            docker_image.push("${IMAGE_TAG}")
            docker_image.push('latest')
          }
        }
      }
    }
    
    stage('Trivy Security Scan') {
      when { 
        expression { return params.DOCKER_PUSH } 
      }
      steps {
        sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
      }
    }
    
    stage('Cleanup Docker Images') {
      when { 
        expression { return params.DOCKER_PUSH } 
      }
      steps {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${IMAGE_NAME}:latest || true"
      }
    }
    
    stage("Trigger CD Pipeline") {
      when { 
        expression { return params.DOCKER_PUSH } 
      }
      environment {
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
      }
      steps {
        script {
          sh """
            curl -v -k --user clouduser:${JENKINS_API_TOKEN} \
            -X POST \
            -H 'cache-control: no-cache' \
            -H 'content-type: application/x-www-form-urlencoded' \
            --data 'IMAGE_TAG=${IMAGE_TAG}' \
            'http://54.206.135.188:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
          """
        }
      }
    }
  }
}
