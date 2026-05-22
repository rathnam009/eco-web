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
        sh "docker build -t ${env.ECR_REPO}:${env.IMAGE_TAG} ."
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-creds'
        ]]) {
          sh """
            aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}
            docker tag ${env.ECR_REPO}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}
            docker push ${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}
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
            IMAGE_URI=${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}

            TASK_DEF=\$(aws ecs describe-task-definition \
              --task-definition ${env.TASK_FAMILY} \
              --region ${env.AWS_REGION})

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
              --region ${env.AWS_REGION}

            aws ecs update-service \
              --cluster ${env.ECS_CLUSTER} \
              --service ${env.ECS_SERVICE} \
              --task-definition ${env.TASK_FAMILY} \
              --force-new-deployment \
              --region ${env.AWS_REGION}
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
              --cluster ${env.ECS_CLUSTER} \
              --services ${env.ECS_SERVICE} \
              --region ${env.AWS_REGION}
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
      // Added 'env.' prefix here to fix the "No such property" crash
      sh "docker rmi ${env.ECR_REPO}:${env.IMAGE_TAG} || true"
      sh "docker rmi ${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG} || true"
    }
  }
}
