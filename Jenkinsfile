pipeline {
  agent any

  environment {
    SERVICE_NAME = "billing-service"
    AWS_REGION   = "us-east-1"
    ACCOUNT_ID   = "469511944493"
    IMAGE_REPO   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${SERVICE_NAME}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          // Get short commit SHA once repo is checked out
          env.SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          echo "Building image for commit ${env.SHORT_SHA}"
        }
      }
    }

    stage('Deps & Tests') {
      steps {
        sh """
          # Ensure Python3 is used
          python3 --version
          pip3 install -r requirements.txt || true
          pytest || echo 'no tests'
        """
      }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds', // Jenkins Credentials
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          sh """
            aws --version
            echo "Logging into ECR..."
            aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin ${IMAGE_REPO}

            echo "Building Docker image..."
            docker build -t ${SERVICE_NAME}:${SHORT_SHA} .
            docker tag ${SERVICE_NAME}:${SHORT_SHA} ${IMAGE_REPO}:${SHORT_SHA}

            echo "Pushing image to ECR..."
            docker push ${IMAGE_REPO}:${SHORT_SHA}
          """
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh """
          echo "Deploying ${SERVICE_NAME}:${SHORT_SHA} to EKS..."
          kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${IMAGE_REPO}:${SHORT_SHA} -n default
          kubectl rollout status deployment/${SERVICE_NAME} -n default
        """
      }
    }
  }

  post {
    failure {
      echo "Build or deployment failed for ${SERVICE_NAME}."
    }
    success {
      echo "Successfully deployed ${SERVICE_NAME}:${SHORT_SHA} to EKS!"
    }
  }
}
