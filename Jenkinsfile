pipeline {
  agent { label 'Jenkins-Agent' }

  tools {
    jdk 'Java17'
    maven 'Maven3'
  }

  parameters {
    booleanParam(name: 'DOCKER_PUSH', defaultValue: false)
    string(name: 'APP_NAME', defaultValue: 'register-app')
    string(name: 'RELEASE', defaultValue: '1.0.0')
    string(name: 'DOCKERHUB_USER', defaultValue: 'your-dockerhub-username')
    string(name: 'DOCKERHUB_CRED_ID', defaultValue: 'dockerhub')
  }

  options {
    timestamps()
  }

  stages {
    stage('Cleanup') {
      steps {
        cleanWs()
      }
    }

    stage('Checkout') {
      steps {
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/mengning-li/register-app.git'
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
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
          archiveArtifacts artifacts: '**/target/*.jar, **/target/*.war', allowEmptyArchive: true, fingerprint: true
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh 'mvn -B sonar:sonar'
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

    stage('Docker Build') {
      when { expression { return params.DOCKER_PUSH } }
      steps {
        script {
          def imageRepo = "${params.DOCKERHUB_USER}/${params.APP_NAME}"
          def imageTag = "${params.RELEASE}-${env.BUILD_NUMBER}"
          def img = docker.build("${imageRepo}:${imageTag}")
          env.IMAGE_REPO = imageRepo
          env.IMAGE_TAG = imageTag
        }
      }
    }

    stage('Docker Push') {
      when { expression { return params.DOCKER_PUSH } }
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', params.DOCKERHUB_CRED_ID) {
            docker.image("${env.IMAGE_REPO}:${env.IMAGE_TAG}").push()
            docker.image("${env.IMAGE_REPO}:${env.IMAGE_TAG}").push('latest')
          }
        }
      }
    }

    stage('Trivy Scan') {
      when { expression { return params.DOCKER_PUSH } }
      steps {
        sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${env.IMAGE_REPO}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
      }
    }

    stage('Local Docker Cleanup') {
      when { expression { return params.DOCKER_PUSH } }
      steps {
        sh "docker rmi ${env.IMAGE_REPO}:${env.IMAGE_TAG} || true"
        sh "docker rmi ${env.IMAGE_REPO}:latest || true"
      }
    }
  }
}
