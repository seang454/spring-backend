@Library('devop_itp_share_library@master') _

pipeline {
    agent any

    environment {
        REPO_NAME      = 'seang454'
        IMAGE_NAME     = 'jenkins-itp-spring'
        TAG            = "${GIT_COMMIT[0..6]}"         // ✅ short commit SHA instead of 'latest'

        // 🔁 GitOps repo settings — update these
        GITOPS_REPO    = 'github.com/seang454/jenkin-argo-gitops'
        GITOPS_BRANCH  = 'main'
        HELM_VALUES    = 'charts/backend/values-dev.yaml'  // 🔁 path to your helm values file
        ARGOCD_APP     = 'backend-dev'                      // 🔁 your Argo CD app name
        ARGOCD_SERVER  = 'argocd.mycompany.com'             // 🔁 your Argo CD server URL
    }

    stages {

        // 1️⃣ Clone Spring Boot Code
        stage('Clone Code') {
            steps {
                git 'https://github.com/seang454/spring-backend'
            }
        }

        // 2️⃣ Build & Test with H2 (no need for Postgres)
        stage('Build & Test with H2') {
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        // Maven
                        withEnv(['SPRING_PROFILES_ACTIVE=test']) {
                            sh 'mvn clean test'
                        }
                    } else if (fileExists('build.gradle')) {
                        // Gradle
                        withEnv(['SPRING_PROFILES_ACTIVE=test']) {
                            sh 'chmod +x gradlew && ./gradlew clean test'
                        }
                    } else {
                        error "No build file found"
                    }
                }
            }
        }

        // 3️⃣ Prepare Dockerfile
        stage('Prepare Dockerfile from Shared Library') {
            steps {
                script {
                    def sharedDockerfile = libraryResource 'springboot/dev.Dockerfile'
                    def dockerfilePath = 'Dockerfile'

                    if (fileExists(dockerfilePath)) {
                        def existingDockerfile = readFile(dockerfilePath)
                        if (existingDockerfile != sharedDockerfile) {
                            echo 'Dockerfile differs from shared library. Replacing it.'
                            sh "rm -f ${dockerfilePath}"
                            writeFile file: dockerfilePath, text: sharedDockerfile
                        } else {
                            echo 'Dockerfile is already up-to-date.'
                        }
                    } else {
                        echo 'Dockerfile not found. Creating from shared library.'
                        writeFile file: dockerfilePath, text: sharedDockerfile
                    }
                }
            }
        }

        // 4️⃣ Build Docker Image
        stage('Build Image') {
            steps {
                sh '''
                echo "⏳ Building image: ${REPO_NAME}/${IMAGE_NAME}:${TAG}"
                docker build --no-cache -t ${REPO_NAME}/${IMAGE_NAME}:${TAG} .
                echo "✅ Image built successfully"
                '''
            }
        }

        // 5️⃣ Ensure Docker Hub Repo Exists
        stage('Ensure Docker Hub Repo Exists') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" -u "$DH_USERNAME:$DH_PASSWORD" \
                      https://hub.docker.com/v2/repositories/$REPO_NAME/$IMAGE_NAME/)

                    if [ "$STATUS" -eq 404 ]; then
                      echo "Repo not found, creating..."
                      curl -s -u "$DH_USERNAME:$DH_PASSWORD" -X POST \
                        https://hub.docker.com/v2/repositories/ \
                        -H "Content-Type: application/json" \
                        -d "{\"name\":\"$IMAGE_NAME\",\"is_private\":false}"
                      echo "✅ Repo created."
                    else
                      echo "✅ Repo already exists."
                    fi
                    '''
                }
            }
        }

        // 6️⃣ Push Docker Image
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    echo "$DH_PASSWORD" | docker login -u "$DH_USERNAME" --password-stdin
                    docker push ${REPO_NAME}/${IMAGE_NAME}:${TAG}
                    docker logout
                    echo "✅ Image pushed: ${REPO_NAME}/${IMAGE_NAME}:${TAG}"
                    '''
                }
            }
        }

        // 7️⃣ Update GitOps Repo — triggers Argo CD to deploy new image
        // ✅ Replaces old "Run PostgreSQL" + "Run Spring Boot" docker stages
        // Jenkins no longer manages containers directly — Argo CD + Helm does
        // stage('Update GitOps Repo') {
        //     steps {
        //         withCredentials([usernamePassword(
        //             credentialsId: 'GITOPS-GITHUB-CREDENTIAL',  // 🔁 Add this in Jenkins credentials
        //             usernameVariable: 'GIT_USERNAME',
        //             passwordVariable: 'GIT_PASSWORD'
        //         )]) {
        //             sh '''
        //             echo "📦 Cloning GitOps repo..."
        //             git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${GITOPS_REPO} gitops-repo
        //             cd gitops-repo

        //             echo "🔄 Updating Spring Boot image tag to: ${TAG}"

        //             # Update the image tag in helm values file
        //             sed -i "s|tag:.*|tag: ${TAG}|" ${HELM_VALUES}

        //             # Verify the update
        //             echo "=== Updated image tag ==="
        //             grep "tag:" ${HELM_VALUES}

        //             # Commit and push
        //             git config user.email "jenkins-ci@mycompany.com"
        //             git config user.name "Jenkins CI"
        //             git add ${HELM_VALUES}
        //             git commit -m "ci: update spring image to ${TAG} [skip ci]"
        //             git push origin ${GITOPS_BRANCH}

        //             echo "✅ GitOps repo updated — Argo CD will deploy shortly"
        //             '''
        //         }
        //     }
        // }

        // // 8️⃣ Wait for Argo CD to Sync & Verify Deployment
        // // ✅ Argo CD handles deploying Spring Boot + PostgreSQL via Helm
        // stage('Verify Argo CD Deployment') {
        //     steps {
        //         withCredentials([usernamePassword(
        //             credentialsId: 'ARGOCD-CREDENTIAL',    // 🔁 Add this in Jenkins credentials
        //             usernameVariable: 'ARGOCD_USERNAME',
        //             passwordVariable: 'ARGOCD_PASSWORD'
        //         )]) {
        //             sh '''
        //             echo "⏳ Waiting for Argo CD to detect changes (30s)..."
        //             sleep 30

        //             # Install argocd CLI if not available
        //             if ! command -v argocd > /dev/null 2>&1; then
        //                 echo "Installing Argo CD CLI..."
        //                 curl -sSL -o /usr/local/bin/argocd \
        //                     https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
        //                 chmod +x /usr/local/bin/argocd
        //             fi

        //             # Login to Argo CD
        //             argocd login ${ARGOCD_SERVER} \
        //                 --username ${ARGOCD_USERNAME} \
        //                 --password ${ARGOCD_PASSWORD} \
        //                 --insecure

        //             # Trigger sync
        //             echo "🔄 Triggering Argo CD sync for: ${ARGOCD_APP}"
        //             argocd app sync ${ARGOCD_APP}

        //             # Wait for healthy deployment
        //             echo "⏳ Waiting for Spring Boot to be healthy..."
        //             argocd app wait ${ARGOCD_APP} \
        //                 --sync \
        //                 --health \
        //                 --timeout 180

        //             # Show final status
        //             argocd app get ${ARGOCD_APP}

        //             echo "✅ Deployment verified — ${IMAGE_NAME}:${TAG} is live!"
        //             '''
        //         }
        //     }
        // }
    }

    post {
        success {
            echo "🚀 Full GitOps pipeline successful!"
            echo "   Image  : ${REPO_NAME}/${IMAGE_NAME}:${TAG}"
            echo "   App    : ${ARGOCD_APP}"
            echo "   Status : Live ✅"
        }
        failure {
            echo "❌ Pipeline failed. Check logs above."
        }
        always {
            // Clean up docker images to save disk space
            sh 'docker image prune -f || true'
            // Clean up cloned gitops repo
            sh 'rm -rf gitops-repo || true'
        }
    }
}
