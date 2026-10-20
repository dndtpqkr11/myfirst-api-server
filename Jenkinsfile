pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    triggers { pollSCM('* * * * *') }
    environment {
        IMAGE_BASE = 'skala-registry.skala-us.com/class-4/sk0115-webserver'
        GITOPS_BRANCH = 'gitops'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def commit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "1.0-ci${env.BUILD_NUMBER}-${commit}"
                }
            }
        }
        stage('Maven Build') {
            steps { sh 'mvn -B -ntp clean package -DskipTests' }
        }
        stage('Image Build and Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'skala-image-registry-id', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                        set -eu
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT
                        printf '%s' "$REG_PASS" | docker login skala-registry.skala-us.com --username "$REG_USER" --password-stdin
                        docker buildx build --platform linux/amd64 --provenance=false \
                            --tag "$IMAGE_BASE:$IMAGE_TAG" --push .
                    '''
                }
            }
        }
        stage('GitOps Update') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'skala-github-id', gitToolName: 'Default')]) {
                    sh '''
                        set -eu
                        set +x
                        GITOPS_DIR="$(mktemp -d)"
                        trap 'git worktree remove --force "$GITOPS_DIR" 2>/dev/null || true' EXIT
                        if git ls-remote --exit-code --heads origin "$GITOPS_BRANCH" >/dev/null; then
                            git fetch origin "refs/heads/$GITOPS_BRANCH:refs/remotes/origin/$GITOPS_BRANCH"
                            git worktree add --detach "$GITOPS_DIR" "origin/$GITOPS_BRANCH"
                        else
                            git worktree add --detach "$GITOPS_DIR" HEAD
                        fi
                        rm -rf "$GITOPS_DIR/kustomize"
                        git archive HEAD kustomize | tar -x -C "$GITOPS_DIR"
                        (cd "$GITOPS_DIR/kustomize/overlays/dev" && \
                            kustomize edit set image "$IMAGE_BASE=$IMAGE_BASE:$IMAGE_TAG")
                        git -C "$GITOPS_DIR" config user.name 'sk0115-gitops'
                        git -C "$GITOPS_DIR" config user.email 'sk0115@users.noreply.github.com'
                        git -C "$GITOPS_DIR" add kustomize
                        if ! git -C "$GITOPS_DIR" diff --cached --quiet; then
                            git -C "$GITOPS_DIR" commit -m "Update image to $IMAGE_TAG"
                            git -C "$GITOPS_DIR" push origin "HEAD:refs/heads/$GITOPS_BRANCH"
                        fi
                    '''
                }
            }
        }
    }
    post {
        always { deleteDir() }
    }
}
