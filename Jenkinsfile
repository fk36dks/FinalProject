pipeline {
  agent any
  environment {
    REGISTRY = "docker.io"
    IMAGE    = "fk36dks/finalproject"     // <<< 하드코딩 (가장 안전)
    TAG      = "1.0-${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
    GIT_USER = "ci-bot"
    GIT_EMAIL= "ci-bot@example.com"
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
          # PyYAML 설치 (yq 없이 파이썬으로 수정)
          pip install --user pyyaml >/dev/null 2>&1 || true
          export PATH="$HOME/.local/bin:$PATH"

          python3 - <<'PY'
import os, yaml
p="deploy/base/kustomization.yaml"
with open(p) as f: data=yaml.safe_load(f)
imgs=data.get("images") or [{}]
if not imgs or not isinstance(imgs[0], dict): imgs=[{}]
imgs[0]["name"]=os.environ["REGISTRY"]+"/"+os.environ["IMAGE"]
imgs[0]["newTag"]=os.environ["TAG"]
data["images"]=imgs
with open(p,"w") as f: yaml.safe_dump(data, f, sort_keys=False)
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
        withCredentials([usernamePassword(credentialsId: 'Git-creds', usernameVariable: 'GITUSER', passwordVariable: 'GITPASS')]) {  // <<< ID 통일
          sh '''
            CURRENT=$(git config --get remote.origin.url)
            AUTH_URL=$(echo "$CURRENT" | sed "s#https://#https://$GITUSER:$GITPASS@#")
            git push "$AUTH_URL" HEAD:main
          '''
        }
      }
    }
  }
  post {
    success { echo "OK: 이미지 푸시 & 매니페스트 태그 업데이트 완료 → ArgoCD가 싱크합니다." }
    failure { echo "FAIL: 콘솔 로그 확인" }
  }
}

