pipeline {
    agent any

    environment {
        GITHUB_USER      = "tinotenda-alfaneti"
        REPO_NAME        = "${env.JOB_NAME.split('/')[1]}"
        IMAGE_REPOSITORY = "ghost"
        IMAGE_TAG        = "6.5"
        IMAGE_DEST       = "${IMAGE_REPOSITORY}:${IMAGE_TAG}"
        APP_NAME         = "ghost"
        RELEASE_NAME     = "ghost"
        NAMESPACE        = "blog"
        KUBECONFIG_CRED  = "kubeconfigglobal"
        PATH             = "$WORKSPACE/bin:$PATH"
        CACHE_DIR        = "${env.JENKINS_HOME}/cache/bin"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "[Checkout] Checking out ${REPO_NAME}..."
                checkout scm
                sh 'mkdir -p $WORKSPACE/bin'
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                    set -euo pipefail

                    echo "[Setup] Preparing tool cache..."
                    CACHE_DIR="${CACHE_DIR:-/var/jenkins_home/cache/bin}"
                    mkdir -p "$CACHE_DIR"

                    # Detect architecture
                    ARCH=$(uname -m)
                    case "$ARCH" in
                        x86_64)  KARCH="amd64" ;;
                        aarch64) KARCH="arm64" ;;
                        armv7l)  KARCH="armv7" ;;
                        *) echo "[Error] Unsupported architecture: $ARCH" >&2; exit 1 ;;
                    esac

                    # kubectl
                    if [ ! -x "$CACHE_DIR/kubectl" ]; then
                        echo "[Setup] Installing kubectl..."
                        VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
                        curl -fsSLo "$CACHE_DIR/kubectl" "https://storage.googleapis.com/kubernetes-release/release/${VER}/bin/linux/${KARCH}/kubectl"
                        chmod +x "$CACHE_DIR/kubectl"
                    else
                        echo "[Setup] Using cached kubectl"
                    fi

                    # Helm
                    if [ ! -x "$CACHE_DIR/helm" ]; then
                        HELM_VER="v3.14.4"
                        echo "[Setup] Installing Helm ${HELM_VER}..."
                        curl -fsSLO "https://get.helm.sh/helm-${HELM_VER}-linux-${KARCH}.tar.gz"
                        tar -zxf "helm-${HELM_VER}-linux-${KARCH}.tar.gz"
                        mv "linux-${KARCH}/helm" "$CACHE_DIR/"
                        chmod +x "$CACHE_DIR/helm"
                        rm -rf "linux-${KARCH}" "helm-${HELM_VER}-linux-${KARCH}.tar.gz"
                    else
                        echo "[Setup] Using cached Helm"
                    fi

                    export PATH="$CACHE_DIR:$PATH"

                    echo "[Info] Tool versions:"
                    "$CACHE_DIR/kubectl" version --client || true
                    "$CACHE_DIR/helm" version || true
                '''
            }
        }

        stage('Lint Chart') {
            steps {
                sh '''
                    export PATH="$CACHE_DIR:$PATH"

                    echo "[Lint] Running helm lint..."
                    "$CACHE_DIR/helm" lint ghost-chart

                    echo "[Lint] Rendering manifests (helm template)..."
                    "$CACHE_DIR/helm" template "${RELEASE_NAME}" ghost-chart > helm-rendered.yaml
                '''
            }
        }

        stage('Verify Cluster Access') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        mkdir -p "$WORKSPACE/.kube"
                        cp "$KUBECONFIG_FILE" "$WORKSPACE/.kube/config"
                        chmod 600 "$WORKSPACE/.kube/config"
                        export KUBECONFIG="$WORKSPACE/.kube/config"
                        export PATH="$CACHE_DIR:$PATH"

                        echo "[Cluster] Verifying cluster connectivity..."
                        "$CACHE_DIR/kubectl" cluster-info
                    '''
                }
            }
        }

        stage('Prepare Namespace') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                sh '''
                    export KUBECONFIG=$WORKSPACE/.kube/config
                    echo "Ensuring namespace ${NAMESPACE} exists..."
                    $CACHE_DIR/kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                    echo "Namespace setup complete for ${NAMESPACE}"
                '''
                }
            }
        }

        stage('Scan Image with Trivy') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$WORKSPACE/.kube/config"
                        export PATH="$CACHE_DIR:$PATH"
                        IMAGE_DEST="${IMAGE_DEST}"

                        echo "[Security] Running Trivy scan for ${IMAGE_DEST}..."
                        sed "s|__IMAGE_DEST__|${IMAGE_DEST}|g" "$WORKSPACE/ci/kubernetes/trivy.yaml" > trivy-job.yaml

                        "$CACHE_DIR/kubectl" delete job trivy-scan -n "${NAMESPACE}" --ignore-not-found=true
                        "$CACHE_DIR/kubectl" apply -f trivy-job.yaml -n "${NAMESPACE}"
                        "$CACHE_DIR/kubectl" wait --for=condition=complete job/trivy-scan -n "${NAMESPACE}" --timeout=10m || true
                        "$CACHE_DIR/kubectl" logs job/trivy-scan -n "${NAMESPACE}" || true
                    '''
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$WORKSPACE/.kube/config"
                        export PATH="$CACHE_DIR:$PATH"

                        echo "[Deploy] Deploying ${APP_NAME} with Helm..."
                        "$CACHE_DIR/helm" upgrade --install "${RELEASE_NAME}" "$WORKSPACE/ghost-chart" \
                            --namespace "${NAMESPACE}" \
                            --create-namespace \
                            --set image.repository="${IMAGE_REPOSITORY}" \
                            --set image.tag="${IMAGE_TAG}" \
                            --wait --timeout 5m
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$WORKSPACE/.kube/config"
                        export PATH="$CACHE_DIR:$PATH"

                        echo "[Verify] Running Helm tests..."
                        "$CACHE_DIR/helm" test "${RELEASE_NAME}" --namespace "${NAMESPACE}" --logs --timeout 2m || true
                        "$CACHE_DIR/kubectl" get pods -n "${NAMESPACE}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "[Result] Pipeline completed successfully."
        }
        failure {
            echo "[Result] Pipeline failed."
        }
        always {
            script {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        if [ -f "$WORKSPACE/.kube/config" ]; then
                            export KUBECONFIG="$WORKSPACE/.kube/config"
                            export PATH="$CACHE_DIR:$PATH"
                            "$CACHE_DIR/kubectl" delete job trivy-scan -n "${NAMESPACE}" --ignore-not-found=true
                        fi
                    '''
                }
            }
            cleanWs()
        }
    }
}
