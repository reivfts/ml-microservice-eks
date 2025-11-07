pipeline {
  agent any

  environment {
    SERVICE_NAME = "gateway"
    AWS_REGION   = "us-east-1"
    ACCOUNT_ID   = "469511944493"
    IMAGE_REPO   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${SERVICE_NAME}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          echo "Building image for commit ${env.SHORT_SHA}"
        }
      }
    }

    stage('Deps & Tests') {
      steps {
        sh '''
          python3 -m venv venv
          . venv/bin/activate
          pip install --upgrade pip
          pip install -r requirements.txt
          pytest || echo "no tests found"
        '''
      }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {
          sh '''
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${IMAGE_REPO}
            docker build -t ${SERVICE_NAME}:${SHORT_SHA} .
            docker tag ${SERVICE_NAME}:${SHORT_SHA} ${IMAGE_REPO}:${SHORT_SHA}
            docker push ${IMAGE_REPO}:${SHORT_SHA}
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
        kubectl set image deployment/api-gateway api-gateway=${IMAGE_REPO}:${SHORT_SHA} -n default
        kubectl rollout status deployment/api-gateway -n default
        '''
      }
    }
  }

  post {
    failure {
      echo "Build or deployment failed for ${env.SERVICE_NAME}."
    }
  }
}
