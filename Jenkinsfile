pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:18-alpine
    command:
    - sleep
    - infinity
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - sleep
    - infinity
  - name: git-updater
    image: alpine/git
    command:
    - sleep
    - infinity
'''
    }
  }

  environment {
    IMAGE_NAME  = '10.100.102.51:30300/edim2525/argocd-demo-app'
    GITOPS_REPO = 'http://10.100.102.51:30300/edim2525/argocd-demo-gitops.git'
  }

  stages {

    stage('Test') {
      steps {
        container('node') {
          sh 'npm ci && npm test'
        }
      }
    }

    stage('Build & Push') {
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(
            credentialsId: 'gitea-registry-creds',
            usernameVariable: 'REG_USER',
            passwordVariable: 'REG_PASS'
          )]) {
            sh '''
              AUTH=$(printf '%s:%s' "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
              mkdir -p /kaniko/.docker
              printf '{"auths":{"10.100.102.51:30300":{"auth":"%s"}}}' "$AUTH" \
                > /kaniko/.docker/config.json
              /kaniko/executor \
                --context=dir://${WORKSPACE} \
                --dockerfile=${WORKSPACE}/Dockerfile \
                --destination=${IMAGE_NAME}:${BUILD_NUMBER} \
                --destination=${IMAGE_NAME}:latest \
                --insecure \
                --skip-tls-verify
            '''
          }
        }
      }
    }

    stage('Update Manifest') {
      steps {
        container('git-updater') {
          withCredentials([usernamePassword(
            credentialsId: 'gitea-git-creds',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'
          )]) {
            sh '''
              git config --global user.email "jenkins@demo.local"
              git config --global user.name "Jenkins"
              git clone http://${GIT_USER}:${GIT_PASS}@10.100.102.51:30300/edim2525/argocd-demo-gitops.git /tmp/gitops
              cd /tmp/gitops
              sed -i "s|image: 10.100.102.51:30300/edim2525/argocd-demo-app:.*|image: 10.100.102.51:30300/edim2525/argocd-demo-app:${BUILD_NUMBER}|" manifests/deployment.yaml
              git add manifests/deployment.yaml
              git commit -m "ci: update image to build ${BUILD_NUMBER}"
              git push http://${GIT_USER}:${GIT_PASS}@10.100.102.51:30300/edim2525/argocd-demo-gitops.git HEAD:main
            '''
          }
        }
      }
    }

  }
}
