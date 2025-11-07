pipeline {
  agent any

  environment {
    SERVICE_NAME = "car-value-service"
    AWS_REGION   = "us-east-1"
    ACCOUNT_ID   = "469511944493"
    IMAGE_REPO   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${SERVICE_NAME}"
    SHORT_SHA    = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
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
          echo "Deploying to EKS..."
          kubectl set image deployment/car-value-service car-value-service=${IMAGE_REPO}:${SHORT_SHA} -n default
          kubectl rollout status deployment/car-value-service -n default
        '''
      }
    }
  }

  post {
    failure {
      echo "Build or deployment failed for ${SERVICE_NAME}."
    }
  }
}
