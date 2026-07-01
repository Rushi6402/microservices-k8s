pipeline {
    agent any

    environment {
        DOCKERHUB_USER   = 'infrawave'       
        REGISTRY         = "docker.io/${DOCKERHUB_USER}"
        GIT_REPO_URL     = 'https://github.com/Rushi6402/microservices-k8s.git' 
        GIT_BRANCH       = 'main'
        VALUES_FILE      = 'helm-chart/values.yaml'
        DOCKERHUB_CREDS  = 'dockerhub-creds'   // Jenkins credential ID
        GITHUB_CREDS     = 'github-creds'      // Jenkins credential ID
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                }
            }
        }

        stage('Detect changed services') {
            steps {
                script {
                    // Compare current commit with previous commit on this branch.
                    // Falls back to building everything if there is no previous commit (first run).
                    def prevCommitExists = sh(script: "git rev-parse HEAD~1", returnStatus: true) == 0

                    def changed
                    if (prevCommitExists) {
                        changed = sh(
                            script: "git diff --name-only HEAD~1 HEAD | grep '^src/' | cut -d'/' -f2 | sort -u || true",
                            returnStdout: true
                        ).trim()
                    } else {
                        changed = sh(
                            script: "ls src | tr '\\n' ' '",
                            returnStdout: true
                        ).trim().replaceAll(' ', '\n')
                    }

                    if (!changed) {
                        echo "No service changes detected. Nothing to build."
                        env.CHANGED_SERVICES = ""
                    } else {
                        env.CHANGED_SERVICES = changed
                        echo "Changed services:\n${changed}"
                    }
                }
            }
        }

        stage('Build & Push images') {
            when { expression { return env.CHANGED_SERVICES?.trim() } }
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DU', passwordVariable: 'DP')]) {
                    sh "echo \$DP | docker login -u \$DU --password-stdin"
                }
                script {
                    def services = env.CHANGED_SERVICES.split('\n')
                    for (svc in services) {
                        svc = svc.trim()
                        if (!svc) continue
                        if (!fileExists("src/${svc}/Dockerfile")) {
                            echo "Skipping ${svc} - no Dockerfile found"
                            continue
                        }
                        def tag = "sha-${env.GIT_COMMIT_SHORT}"
                        def image = "${REGISTRY}/${svc}:${tag}"
                        echo "Building ${image}"
                        sh "docker build -t ${image} ./src/${svc}"
                        sh "docker push ${image}"
                        env."TAG_${svc}" = tag
                    }
                }
            }
        }

        stage('Update Helm values.yaml') {
           when { expression { return env.CHANGED_SERVICES?.trim() } }
           steps {
              script {
                 def services = env.CHANGED_SERVICES.split('\n')
                 for (svc in services) {
                     svc = svc.trim()
                     if (!svc) continue
                     def tagVar = env."TAG_${svc}"
                     if (!tagVar) continue
                     sh """
                          yq e -i '.images.${svc}.repository = \"${REGISTRY}/${svc}\"' ${VALUES_FILE}
                          yq e -i '.images.${svc}.tag = \"${tagVar}\"' ${VALUES_FILE}
                    """
                }
             }
         }
      }

        stage('Commit & Push updated values.yaml') {
            when { expression { return env.CHANGED_SERVICES?.trim() } }
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDS}", usernameVariable: 'GU', passwordVariable: 'GP')]) {
                    sh """
                        git config user.email "jenkins@ci.local"
                        git config user.name "jenkins"
                        git add ${VALUES_FILE}
                        git commit -m "ci: update image tag(s) [skip ci]" || echo "Nothing to commit"
                        git push https://\$GU:\$GP@\$(echo ${GIT_REPO_URL} | sed 's#https://##') HEAD:${GIT_BRANCH}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline finished. ArgoCD will detect the Git change and sync automatically."
        }
        failure {
            echo "Pipeline failed - check logs above."
        }
    }
}
