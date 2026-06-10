pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        APP_NAME = "dssp-jsp-test-10-06".toLowerCase().trim()
        DOCKER_IMAGE = "chaitanyapandeygspann/${APP_NAME}"
        DOCKER_TAG = "1.0.${BUILD_NUMBER}"
        IMAGE_TAG = "${DOCKER_IMAGE}:${DOCKER_TAG}"
        GITOPS_REPO = "https://github.com/BackstageSSPPoC/k8s-manifests.git"
        APP_PORTS = "8080"
        DEPLOY_ENV = "dev"       
        DEPLOY_NAMESPACE = "dev"
        TARGET_CLUSTER  = "dev-dssp-dev-cluster-cluster-10"
        ARGOCD_SERVER   = '35.184.124.65'
        TFSTATE_BUCKET  = 'gssp-tfstate-poc-2026'
        MAIN_CONTEXT    = 'gke_gspann-backstage_us-central1_backstage-poc-autopilot'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Decide Pipeline Flow') {
            steps {
                script {
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "GIT_BRANCH: ${env.GIT_BRANCH}"

                    if (env.BRANCH_NAME == "main" || env.BRANCH_NAME.endsWith("/main") || env.GIT_BRANCH?.endsWith("main")) {
                        echo "Main branch detected → CI + CD"
                        env.RUN_MODE = "cd"
                    } else {
                        echo "Non-main branch → CI only"
                        env.RUN_MODE = "ci"
                    }
                }
            }
        }


// ================= CI STAGES =================

        stage('Build & Test') {
            steps {
                sh 'mvn clean verify -B'
                // -B = batch mode (no interactive prompts, clean Jenkins output)
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn sonar:sonar \
                      -Dsonar.projectKey=${APP_NAME} \
                      -Dsonar.sources=src/main/java \
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

// ================= CD STAGES =================

        stage('Build Docker Image') {
            when {
                expression { env.RUN_MODE == "cd" }
            }
            steps {
                sh 'docker build -t ${IMAGE_TAG} .'
            }
        }

        stage('Login to Docker Hub') {
            when {
                expression { env.RUN_MODE == "cd" }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { env.RUN_MODE == "cd" }
            }
            steps {
                sh 'docker push ${IMAGE_TAG}'
            }
        }

        stage('Update GitOps Repo') {
            when {
                expression { env.RUN_MODE == "cd" }
            }
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    echo "====================================="
                    echo "Fetching target cluster server from ArgoCD..."

                    kubectl config use-context ${MAIN_CONTEXT}

                    ARGOCD_PASSWORD=$(kubectl get secret argocd-initial-admin-secret \
                      -n argocd \
                      -o jsonpath="{.data.password}" | base64 -d)

                    argocd login ${ARGOCD_SERVER} \
                      --username admin \
                      --password $ARGOCD_PASSWORD \
                      --insecure

                    if [ "${TARGET_CLUSTER}" = "backstage-poc-autopilot" ]; then
                      TARGET_CLUSTER_SERVER="https://kubernetes.default.svc"
                    else
                      TARGET_CLUSTER_SERVER=$(argocd cluster list -o json | \
                        python3 -c "
import json, sys
clusters = json.load(sys.stdin)
for c in clusters:
    if c.get('name') == '${TARGET_CLUSTER}':
        print(c.get('server', ''))
        break
")
                    fi

                    if [ -z "$TARGET_CLUSTER_SERVER" ]; then
                      echo "WARNING: Cluster '${TARGET_CLUSTER}' not found in ArgoCD."
                      echo "Falling back to main cluster."
                      TARGET_CLUSTER_SERVER="https://kubernetes.default.svc"
                    fi

                    echo "Target Cluster : ${TARGET_CLUSTER}"
                    echo "Target Server  : $TARGET_CLUSTER_SERVER"
                    echo "====================================="

                    rm -rf k8s-manifests

                    git clone --depth 1 https://${GITHUB_TOKEN}@github.com/BackstageSSPPoC/k8s-manifests.git
                    cd k8s-manifests

                    mkdir -p apps/${APP_NAME}/${DEPLOY_ENV}

                    mkdir -p argocd/

                    cp ../manifest-templates/application.yaml argocd/${APP_NAME}-${DEPLOY_ENV}.yaml || true

                    cp ../manifest-templates/deployment.yaml apps/${APP_NAME}/${DEPLOY_ENV}/ || true
                    cp ../manifest-templates/service.yaml apps/${APP_NAME}/${DEPLOY_ENV}/ || true
                    cp ../manifest-templates/ingress.yaml apps/${APP_NAME}/${DEPLOY_ENV}/ || true

                    sed -i "s|\\${APP_NAME}|${APP_NAME}|g" apps/${APP_NAME}/${DEPLOY_ENV}/*.yaml || true
                    sed -i "s|\\${DOCKER_IMAGE}|${IMAGE_TAG}|g" apps/${APP_NAME}/${DEPLOY_ENV}/*.yaml || true
                    sed -i "s|\\${APP_PORT}|${APP_PORTS}|g" apps/${APP_NAME}/${DEPLOY_ENV}/*.yaml || true
                    sed -i "s|\\${NAMESPACE}|${DEPLOY_NAMESPACE}|g" apps/${APP_NAME}/${DEPLOY_ENV}/*.yaml || true
                    sed -i "s|\\${APP_NAME}|${APP_NAME}|g" argocd/${APP_NAME}-${DEPLOY_ENV}.yaml || true
                    sed -i "s|\\${NAMESPACE}|${DEPLOY_NAMESPACE}|g" argocd/${APP_NAME}-${DEPLOY_ENV}.yaml || true
                    sed -i "s|\\${TARGET_CLUSTER_SERVER}|$TARGET_CLUSTER_SERVER|g" argocd/${APP_NAME}-${DEPLOY_ENV}.yaml || true

                    git config user.email "jenkins@local"
                    git config user.name "jenkins"
                    git add .
                    git commit -m "[${DEPLOY_ENV}] Deploy ${APP_NAME} build ${BUILD_NUMBER} → cluster:${TARGET_CLUSTER}" || echo "No changes to commit"
                    git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
            sh "docker image prune -f || true"
        }
    }
}
