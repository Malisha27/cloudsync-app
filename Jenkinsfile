pipeline {
  agent any

  environment {
    IMAGE       = "27malisha/cloudsync-app"
    GITOPS_REPO = "github.com/Malisha27/cloudsync-gitops.git"
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set Tag') {
      steps {
        script {
          env.TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Build') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }

    stage('Test') {
      steps { sh 'docker run --rm $IMAGE:$TAG python -m pytest -q' }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                         usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh 'echo $DH_TOKEN | docker login -u $DH_USER --password-stdin'
          sh 'docker push $IMAGE:$TAG'
        }
      }
    }

    stage('Update GitOps Repo') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds',
                         usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            rm -rf gitops
            git clone https://$GH_USER:$GH_TOKEN@$GITOPS_REPO gitops
            cd gitops
            sed -i "s|image: .*cloudsync-app:.*|image: $IMAGE:$TAG|" apps/cloudsync/deployment.yaml
            git config user.email "jenkins@cloudsync.local"
            git config user.name "Jenkins CI"
            git add apps/cloudsync/deployment.yaml
            git commit -m "Deploy cloudsync-app:$TAG" || echo "No changes to commit"
            git push origin main
          '''
        }
      }
    }
  }
}