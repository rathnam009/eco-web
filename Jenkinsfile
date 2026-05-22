pipeline {
  agent any

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '651770526874.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO     = 'my-frontend'
    ECS_CLUSTER  = 'my-frontend-cluster'
    ECS_SERVICE  = 'my-frontend-service'
    TASK_FAMILY  = 'my-frontend-task'
    IMAGE_TAG    = "${env.BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Docker Build') {
      steps {
        sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds'
        ]]) {
          sh """
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
            docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
            docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Deploy to ECS') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds'
        ]]) {
          sh """
            IMAGE_URI=${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}

            TASK_DEF=\$(aws ecs describe-task-definition \
              --task-definition ${TASK_FAMILY} \
              --region ${AWS_REGION})

            NEW_TASK_DEF=\$(echo \$TASK_DEF | python3 -c "
import sys, json
t = json.load(sys.stdin)['taskDefinition']
t['containerDefinitions'][0]['image'] = '\$IMAGE_URI'
for k in ['taskDefinitionArn','revision','status',
          'requiresAttributes','compatibilities',
          'registeredAt','registeredBy']:
    t.pop(k, None)
print(json.dumps(t))")

            aws ecs register-task-definition \
              --cli-input-json "\$NEW_TASK_DEF" \
              --region ${AWS_REGION}

            aws ecs update-service \
              --cluster ${ECS_CLUSTER} \
              --service ${ECS_SERVICE} \
              --task-definition ${TASK_FAMILY} \
              --force-new-deployment \
              --region ${AWS_REGION}
          """
        }
      }
    }

    stage('Wait for Deployment') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds'
        ]]) {
          sh """
            aws ecs wait services-stable \
              --cluster ${ECS_CLUSTER} \
              --services ${ECS_SERVICE} \
              --region ${AWS_REGION}
          """
        }
      }
    }

  }

  post {
    success {
      echo "Deployment successful - Build #${env.BUILD_NUMBER}"
    }
    failure {
      echo "Deployment failed - Build #${env.BUILD_NUMBER}"
    }
    always {
      sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
      sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} || true"
    }
  }
}
