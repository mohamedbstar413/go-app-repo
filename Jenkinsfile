pipeline {
    agent none
    
    environment {
        APP_NAME = 'senior-go'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USERNAME_PLACEHOLDER = 'bstar999'  // Will be overridden by credentials
        VERSION = "${env.BUILD_NUMBER}"
        GO111MODULE = 'on'
        CGO_ENABLED = '0'
        GOOS = 'linux'
        GOARCH = 'amd64'
        DEPLOY_NAMESPACE = 'default'
    }
    
    stages {
        stage('Checkout') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccount: jenkins
  securityContext:
    fsGroup: 1000
  containers:
  - name: git
    image: alpine/git:latest
    command:
    - cat
    tty: true
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
"""
                }
            }
            steps {
                container('git') {
                    checkout scm
                    script {
                        env.GIT_COMMIT_SHORT = sh(
                            script: "git rev-parse --short HEAD",
                            returnStdout: true
                        ).trim()
                        env.IMAGE_TAG = "${VERSION}-${GIT_COMMIT_SHORT}"
                        echo "Git Commit: ${env.GIT_COMMIT_SHORT}"
                        echo "Image Tag: ${env.IMAGE_TAG}"
                    }
                }
            }
        }
        stage('Build & Push Docker Image') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccount: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - sleep
    args:
    - 99999
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    emptyDir: {}
"""
                }
            }
            steps {
                container('kaniko') {
                    checkout scm
                    script {
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            def dockerImage = "${DOCKER_REGISTRY}/${DOCKER_USER}/${APP_NAME}"
                            
                            sh """
                                echo "Creating Docker config for authentication..."
                                
                                # Create base64 encoded auth string
                                AUTH=\$(echo -n "${DOCKER_USER}:${DOCKER_PASS}" | base64 | tr -d '\n')
                                
                                # Create proper Docker config.json
                                cat > /kaniko/.docker/config.json <<EOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "\${AUTH}"
    }
  }
}
EOF
                                
                                echo "Docker config created successfully"
                                
                                echo "Building and pushing Docker image..."
                                echo "Image: ${dockerImage}:${IMAGE_TAG}"
                                
                                /kaniko/executor \
                                    --context ${WORKSPACE} \
                                    --dockerfile ${WORKSPACE}/Dockerfile \
                                    --destination ${dockerImage}:${IMAGE_TAG} \
                                    --destination ${dockerImage}:latest \
                                    --cache=true \
                                    --compressed-caching=false \
                                    --verbosity=info
                                    -snapshot-mode=redo \
                                    --use-new-run \
                                    --push-retry 3 \
                                
                                echo "✅ Image pushed successfully!"
                                echo "  ${dockerImage}:${IMAGE_TAG}"
                                echo "  ${dockerImage}:latest"
                            """
                            
                            env.DOCKER_IMAGE = dockerImage
                        }
                    }
                }
            }
        }
        stage('Create Helm Chart') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccount: jenkins
  containers:
  - name: helm
    image: alpine/helm:3.13.1
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('helm') {
                    sh '''
                        echo "Creating Helm chart from YAML files..."
                        
                        # Create chart structure
                        helm create ${APP_NAME}
                        rm -rf ${APP_NAME}/templates/*
                        
                        # Copy your YAML files (adjust path based on your repo structure)
                        if [ -d "k8s" ]; then
                            echo "Copying from k8s/ directory..."
                            cp k8s/*.yaml ${APP_NAME}/templates/
                        elif [ -d "kubernetes" ]; then
                            echo "Copying from kubernetes/ directory..."
                            cp kubernetes/*.yaml ${APP_NAME}/templates/
                        elif [ -d "manifests" ]; then
                            echo "Copying from manifests/ directory..."
                            cp manifests/*.yaml ${APP_NAME}/templates/
                        else
                            echo "No k8s directory found, looking for YAML files in root..."
                            find . -maxdepth 1 -name "*.yaml" -o -name "*.yml" | grep -v "Chart.yaml" | xargs -I {} cp {} ${APP_NAME}/templates/ || true
                        fi
                        
                        # Update Chart.yaml
                        cat > ${APP_NAME}/Chart.yaml <<EOF
apiVersion: v2
name: ${APP_NAME}
description: A Helm chart for ${APP_NAME} Go application
type: application
version: 1.0.${BUILD_NUMBER}
appVersion: "${IMAGE_TAG}"
maintainers:
  - name: Jenkins
    email: jenkins@example.com
EOF
                        
                        # Create values.yaml
                        cat > ${APP_NAME}/values.yaml <<EOF
replicaCount: 2

image:
  repository: ${DOCKER_IMAGE}
  tag: ${IMAGE_TAG}
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000
  targetPort: 8000

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
EOF
                        
                        # Show what we packaged
                        echo "=== Helm Chart Contents ==="
                        ls -la ${APP_NAME}/templates/
                        
                        # Validate and package
                        helm lint ${APP_NAME}
                        helm package ${APP_NAME}
                        
                        echo "✅ Helm chart created successfully!"
                        ls -la *.tgz
                    '''
                    
                    archiveArtifacts artifacts: '*.tgz', fingerprint: true
                }
            }
        }

        stage('Deploy Helm Chart') {
            when {
                branch 'main'
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccount: jenkins
  containers:
  - name: helm
    image: alpine/helm:3.13.1
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('helm') {
                    sh '''
                        echo "Deploying Helm chart to Kubernetes..."
                        
                        # Deploy or upgrade the release
                        helm upgrade --install ${APP_NAME} \
                            ./${APP_NAME}-1.0.${BUILD_NUMBER}.tgz \
                            --namespace ${DEPLOY_NAMESPACE} \
                            --create-namespace \
                            --wait \
                            --timeout 5m \
                            --set image.tag=${IMAGE_TAG} \
                            --set image.repository=${DOCKER_IMAGE} \
                            --atomic \
                            --cleanup-on-fail
                        
                        echo "✅ Deployment successful!"
                        
                        # Show release info
                        helm list -n ${DEPLOY_NAMESPACE}
                        helm status ${APP_NAME} -n ${DEPLOY_NAMESPACE}
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            when {
                branch 'main'
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccount: jenkins
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('kubectl') {
                    sh '''
                        echo "Verifying deployment..."
                        
                        kubectl get all -n ${DEPLOY_NAMESPACE} -l app.kubernetes.io/name=${APP_NAME}
                        
                        # Wait for rollout
                        kubectl rollout status deployment/${APP_NAME} -n ${DEPLOY_NAMESPACE} --timeout=5m || true
                        
                        # Get pod logs
                        echo "=== Recent Pod Logs ==="
                        kubectl logs -n ${DEPLOY_NAMESPACE} -l app.kubernetes.io/name=${APP_NAME} --tail=50 || true
                        
                        echo "✅ Verification complete!"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ Pipeline completed successfully!"
            echo "Image: ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        always {
            echo "Pipeline execution finished."
        }
    }
}