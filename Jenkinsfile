pipeline {
  agent any
  environment {
    REGISTRY="docker.io"
    IMAGE="${DOCKERHUB_USER}/finalproject"
    TAG="1.0-${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
    GIT_USER="ci-bot"
    GIT_EMAIL="ci-bot@example.com"
  }
  stages {
    stage('Checkout'){ steps { checkout scm } }

    stage('Deps/Smoke'){
      steps { dir('app'){ sh 'python -V || true' } }
    }

    stage('Docker Login'){
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]){
          sh 'echo "$PASS" | docker login -u "$USER" --password-stdin $REGISTRY'
        }
      }
    }

    stage('Build Image'){ steps { sh 'docker build -t $REGISTRY/$IMAGE:$TAG ./app' } }

    stage('Push Image'){ steps { sh 'docker push $REGISTRY/$IMAGE:$TAG' } }

    stage('Bump Kustomize & Commit'){
      steps {
        sh '''
          python3 - <<'PY'
import os,yaml
p="deploy/base/kustomization.yaml"
with open(p) as f: data=yaml.safe_load(f)
imgs=data.get("images") or [{}]
if not imgs or not isinstance(imgs[0],dict): imgs=[{}]
imgs[0]["name"]=os.environ["REGISTRY"]+"/"+os.environ["IMAGE"]
imgs[0]["newTag"]=os.environ["TAG"]
data["images"]=imgs
with open(p,"w") as f: yaml.safe_dump(data,f,sort_keys=False)
PY
          git config user.name "$GIT_USER"
          git config user.email "$GIT_EMAIL"
          git add deploy/base/kustomization.yaml
          git commit -m "ci: bump image tag to $TAG" || echo "no changes"
        '''
      }
    }

    stage('Push to Git'){
      steps {
        withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GITUSER', passwordVariable: 'GITPASS')]){
          sh '''
            CURRENT=$(git config --get remote.origin.url)
            AUTH_URL=$(echo "$CURRENT" | sed "s#https://#https://$GITUSER:$GITPASS@#")
            git push "$AUTH_URL" HEAD:main
          '''
        }
      }
    }
  }
  post { success { echo "OK: 푸시 완료 → ArgoCD 싱크 대기" } }
}
