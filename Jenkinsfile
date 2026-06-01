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
'''
        }
    }
    
    environment {
        REGISTRY   = "harbor.devops-tools.svc.cluster.local:80" // Cluster-internal Harbor URL
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
                // Force this step to run inside the Maven container which has Java runtime
                container('maven') {
                    sonarScan(projectKey: 'java-knative-app', projectName: 'Java-Knative-App')
                }
            }
        }
        
        stage('Container Build') {
            steps {
                // Force this step to run inside the Kaniko container to build without docker.sock
                container('kaniko') {
                    echo "🚀 Building and pushing container image to Harbor..."
                    // Call your shared library image builder task here
                    buildImage(imageName: "${REGISTRY}/${IMAGE_NAME}", tag: "${IMAGE_TAG}")
                }
            }
        }
        
        stage('Update Git Manifest (GitOps Push)') {
            steps {
                // Running back inside default/maven container to execute git mutations
                container('maven') {
                    echo "📝 Updating GitOps Image Tag target to ${IMAGE_TAG}..."
                    sh """
                        sed -i "s|image: .*|image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g" k8s-manifests/knative-service.yaml
                        git config user.email "jenkins-automation@local.com"
                        git config user.name "Jenkins CI"
                        git add k8s-manifests/knative-service.yaml
                        git commit -m "chore: bumped application image tag to version ${IMAGE_TAG} [skip ci]"
                        git push origin main
                    """
                }
            }
        }
    }
}