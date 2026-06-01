// java-knative-app/Jenkinsfile
@Library('jenkins-shared-library@main') _

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: jenkins-build-agent
spec:
  containers:
  - name: maven
    image: maven:3.9.6-eclipse-temurin-17
    command: ['cat']
    tty: true
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ['cat']
    tty: true
    volumeMounts:
    - name: registry-auth
      mountPath: /kaniko/.docker
  volumes:
  - name: registry-auth
    secret:
      secretName: harbor-creds
      items:
      - key: .dockerconfigjson
        path: config.json
'''
        }
    }
    
    environment {
        REGISTRY   = "harbor-core.devops-tools.svc.cluster.local:80" // Corrected internal endpoint
        IMAGE_NAME = "apps/java-knative-service"
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Quality Gate') {
            steps {
                container('maven') {
                    sonarScan(projectKey: 'java-knative-app', projectName: 'Java-Knative-App')
                }
            }
        }
        
        stage('Container Build') {
            steps {
                container('kaniko') {
                    echo "🚀 Building and pushing container image to Harbor..."
                    buildImage(imageName: "${REGISTRY}/${IMAGE_NAME}", tag: "${IMAGE_TAG}")
                }
            }
        }
        
        stage('Update Git Manifest (GitOps Push)') {
            steps {
                container('maven') {
                    echo "📝 Updating GitOps Image Tag target to ${IMAGE_TAG}..."
                    
                    // Explicitly tracking and establishing the Git workspace context
                    sh """
                        git init
                        git config user.email "jenkins-automation@local.com"
                        git config user.name "Jenkins CI"
                        
                        # Apply the tag substitution modification
                        sed -i "s|image: .*|image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g" k8s-manifests/knative-service.yaml
                        
                        # Stage, commit, and push back up to the tracking branch
                        git add k8s-manifests/knative-service.yaml
                        git commit -m "chore: bumped application image tag to version ${IMAGE_TAG} [skip ci]" || echo "No changes to commit"
                        
                        # Using the implicit build token credentials environment to authorize the push
                        git push origin HEAD:main
                    """
                }
            }
        }
    }
}