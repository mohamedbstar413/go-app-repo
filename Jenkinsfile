pipeline {
    agent none
    
    environment {
        APP_NAME = 'senior-go'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${APP_NAME}"
        VERSION = "${env.BUILD_NUMBER}"
        GO111MODULE = 'on'
        CGO_ENABLED = '0'
        GOOS = 'linux'
        GOARCH = 'amd64'
        // Deployment namespace
        DEPLOY_NAMESPACE = 'default'  // or 'default', 'staging', etc.
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
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )]) {
                            sh '''
                                echo "Creating Docker config..."
                                echo "{\\"auths\\":{\\"${DOCKER_REGISTRY}\\":{\\"username\\":\\"${DOCKER_USERNAME}\\",\\"password\\":\\"${DOCKER_PASSWORD}\\"}}}" > /kaniko/.docker/config.json
                                
                                echo "Building and pushing Docker image with Kaniko..."
                                /kaniko/executor \
                                    --context ${WORKSPACE} \
                                    --dockerfile ${WORKSPACE}/Dockerfile \
                                    --destination ${DOCKER_IMAGE}:${IMAGE_TAG} \
                                    --destination ${DOCKER_IMAGE}:latest \
                                    --cache=true \
                                    --compressed-caching=false
                                
                                echo "Image pushed: ${DOCKER_IMAGE}:${IMAGE_TAG}"
                            '''
                        }
                    }
                }
            }
        }
        

        /*stage('Deploy to Kubernetes') {
            when {
                branch 'main'
            }
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
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
"""
                }
            }
            steps {
                container('kubectl') {
                    sh '''
                        echo "Deploying to Kubernetes namespace: ${DEPLOY_NAMESPACE}..."
                        
                        # Check if we have access to the namespace
                        kubectl auth can-i get deployments -n ${DEPLOY_NAMESPACE} || \
                            (echo "ERROR: No permission to access namespace ${DEPLOY_NAMESPACE}" && exit 1)
                        
                        # Update deployment with new image
                        kubectl set image deployment/${APP_NAME} \
                            ${APP_NAME}=${DOCKER_IMAGE}:${IMAGE_TAG} \
                            -n ${DEPLOY_NAMESPACE}
                        
                        # Wait for rollout to complete
                        kubectl rollout status deployment/${APP_NAME} -n ${DEPLOY_NAMESPACE} --timeout=5m
                        
                        echo "Deployment completed successfully!"
                        kubectl get pods -n ${DEPLOY_NAMESPACE} -l app=${APP_NAME}
                    '''
                }
            }
        }*/
        
        /*stage('Smoke Tests') {
            when {
                branch 'main'
            }
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
  - name: curl
    image: curlimages/curl:latest
    command:
    - cat
    tty: true
"""
                }
            }
            steps {
                container('curl') {
                    sh '''
                        echo "Running smoke tests..."
                        
                        # Wait a bit for service to be ready
                        sleep 10
                        
                        # Basic health check using the service in the deployment namespace
                        curl -f http://${APP_NAME}.${DEPLOY_NAMESPACE}.svc.cluster.local:8080/health || exit 1
                        
                        echo "Smoke tests passed!"
                    '''
                }
            }
        }*/
    }
    
    post {
        success {
            echo "Pipeline completed successfully! 🎉"
            echo "Image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed! ❌"
        }
        always {
            cleanWs()
        }
    }
}