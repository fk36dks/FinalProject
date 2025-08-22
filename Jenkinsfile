pipeline {
  agent any

  options {
    // 중복 실행 방지
    disableConcurrentBuilds()
    // 기본 체크아웃 유지(Declarative: Checkout SCM)
    // skipDefaultCheckout(true)  // ← 직접 체크아웃 할 거면 주석 해제
  }

  environment {
    REGISTRY    = 'docker.io'
    DOCKER_USER = 'fk36dks'              // ← 본인 Docker Hub ID
    IMAGE       = "${DOCKER_USER}/finalproject"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          // 짧은 SHA와 태그 생성 (예: 1.0-12-a1b2c3d)
          env.SHORTSHA = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
          env.TAG = "1.0-${env.BUILD_NUMBER}-${env.SHORTSHA}"
        }
      }
    }

    stage('Deps/Smoke') {
      steps {
        dir('app') {
          sh '''
            python -V
            [ -f requirements.txt ] && pip install -r requirements.txt || true
          '''
        }
      }
    }

    stage('Docker Login') {
      steps {
        withCredentials([string(credentialsId: 'dockerhub-creds', variable: 'PASS')]) {
          sh '''
            echo "$PASS" | docker login -u ${DOCKER_USER} --password-stdin ${REGISTRY}
          '''
        }
      }
    }

    stage('Build Image') {
      steps {
        sh '''
          docker build -t ${REGISTRY}/${IMAGE}:${TAG} ./app
        '''
      }
    }

    stage('Push Image') {
      steps {
        sh '''
          docker push ${REGISTRY}/${IMAGE}:${TAG}
        '''
      }
    }

    stage('Bump Kustomize & Commit') {
      steps {
        sh '''
          set -euo pipefail
          # PyYAML 설치(없으면)
          pip3 install --user pyyaml >/dev/null 2>&1 || true

          # kustomization.yaml의 images 섹션 갱신
          python3 - <<'PY'
import os, yaml, pathlib
p = pathlib.Path('deploy/base/kustomization.yaml')
data = {}
if p.exists():
    with p.open() as f:
        data = yaml.safe_load(f) or {}
imgs = data.get('images') or [{}]
if not imgs:
    imgs = [{}]
imgs[0]['name']   = f"{os.environ['REGISTRY']}/{os.environ['IMAGE']}"
imgs[0]['newTag'] = os.environ['TAG']
data['images'] = imgs
with p.open('w') as f:
    yaml.safe_dump(data, f, sort_keys=False)
PY

          # 여기서 커밋은 'ci-bot' 사용자로, 메시지에 [skip ci] 포함
          git config user.name  "ci-bot"
          git config user.email "ci-bot@example.com"
          git add deploy/base/kustomization.yaml
          git commit -m "[skip ci] ci: bump image tag to ${TAG}" || echo "no change"
        '''
      }
    }

    stage('Push to Git') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'Git-creds', usernameVariable: 'GITUSER', passwordVariable: 'GITPASS')]) {
          sh '''
            set -euo pipefail
            CURRENT=$(git config --get remote.origin.url)
            AUTH_URL=$(echo "$CURRENT" | sed "s#https://#https://${GITUSER}:${GITPASS}@#")
            git push "$AUTH_URL" HEAD:main
          '''
        }
      }
    }
  }

  post {
    success {
      echo 'OK: 이미지 푸시 & 태그 커밋 완료 → Argo CD가 자동 싱크합니다.'
    }
    failure {
      echo 'FAIL: 콘솔 로그 확인'
    }
  }
}
