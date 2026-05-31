// java-knative-app/Jenkinsfile
@NonCPS
@Library('jenkins-shared-library@main') _

pipeline {
    agent any
    
    environment {
        REGISTRY   = "harbor.devops-tools.svc.cluster.local" // Cluster-internal Harbor URL
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
                // Invoking shared library step
                sonarScan()
            }
        }
        
        stage('Container Build') {
            steps {
                // Invoking shared library step
                buildImage(imageName: "${REGISTRY}/${IMAGE_NAME}", tag: "${IMAGE_TAG}")
            }
        }
        
        stage('Update Git Manifest (GitOps Push)') {
            steps {
                echo "📝 Updating GitOps Image Tag target to ${IMAGE_TAG}..."
                // Sh script to update k8s manifests via a git push back to the repo
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