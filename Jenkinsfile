pipeline {
    agent any

    environment {
        DOCKERHUB_USER   = 'infrawave'
        REGISTRY         = "docker.io/${DOCKERHUB_USER}"
        GIT_REPO_URL     = 'https://github.com/Rushi6402/microservices-k8s.git'
        GIT_BRANCH       = 'main'
        VALUES_FILE      = 'helm-chart/values.yaml'
        DOCKERHUB_CREDS  = 'dockerhub-creds'
        GITHUB_CREDS     = 'github-creds'
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
                    def prevCommitExists = sh(script: "git rev-parse HEAD~1", returnStatus: true) == 0
                    def changed
                    if (prevCommitExists) {
                        changed = sh(
                            script: "git diff --name-only HEAD~1 HEAD | grep '^src/' | cut -d'/' -f2 | sort -u || true",
                            returnStdout: true
                        ).trim()
                    } else {
                        changed = sh(script: "ls src | tr '\\n' '\\n'", returnStdout: true).trim()
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
                    def skipServices = ['adservice', 'loadgenerator']
                    def services = env.CHANGED_SERVICES.split('\n')
                    def builtServices = []

                    for (svc in services) {
                        svc = svc.trim()
                        if (!svc) continue
                        if (skipServices.contains(svc)) {
                            echo "Skipping ${svc} - excluded from build list"
                            continue
                        }
                        if (!fileExists("src/${svc}/Dockerfile")) {
                            echo "Skipping ${svc} - no Dockerfile found"
                            continue
                        }
                        def firstLine = sh(script: "head -1 src/${svc}/Dockerfile", returnStdout: true).trim()
                        if (firstLine.startsWith('#') && firstLine.contains('trigger')) {
                            echo "Skipping ${svc} - Dockerfile corrupted: ${firstLine}"
                            continue
                        }
                        try {
                            def tag = "sha-${env.GIT_COMMIT_SHORT}"
                            def image = "${REGISTRY}/${svc}:${tag}"
                            echo "Building ${image}"
                            sh "docker build -t ${image} ./src/${svc}"
                            sh "docker push ${image}"
                            env."TAG_${svc}" = tag
                            builtServices.add(svc)
                            echo "Built and pushed ${image}"
                        } catch (err) {
                            echo "WARNING: Failed to build ${svc}: ${err.message} - continuing"
                        }
                    }

                    if (builtServices.isEmpty()) {
                        echo "No services were successfully built"
                    } else {
                        echo "Successfully built: ${builtServices.join(', ')}"
                        env.BUILT_SERVICES = builtServices.join(',')
                    }
                }
            }
        }

        stage('Update Helm values.yaml') {
            when {
                expression { return env.CHANGED_SERVICES?.trim() && env.BUILT_SERVICES?.trim() }
            }
            steps {
                script {
                    def builtServices = env.BUILT_SERVICES.split(',')
                    for (svc in builtServices) {
                        svc = svc.trim()
                        if (!svc) continue
                        def tagVar = env."TAG_${svc}"
                        if (!tagVar) continue
                        echo "Updating values.yaml for ${svc} -> ${tagVar}"
                        sh "yq e -i '.images.${svc}.repository = \"${REGISTRY}/${svc}\"' ${VALUES_FILE}"
                        sh "yq e -i '.images.${svc}.tag = \"${tagVar}\"' ${VALUES_FILE}"
                    }
                }
            }
        }

        stage('Commit & Push updated values.yaml') {
            when {
                expression { return env.CHANGED_SERVICES?.trim() && env.BUILT_SERVICES?.trim() }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDS}", usernameVariable: 'GU', passwordVariable: 'GP')]) {
                    sh """
                        git config user.email "jenkins@ci.local"
                        git config user.name "jenkins"
                        git add ${VALUES_FILE}
                        git diff --cached --quiet && echo "Nothing to commit" || git commit -m "ci: update image tags [skip ci]"
                        git push https://\$GU:\$GP@\$(echo ${GIT_REPO_URL} | sed 's#https://##') HEAD:${GIT_BRANCH}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS. Built: ${env.BUILT_SERVICES ?: 'none'}. ArgoCD will auto-sync."
        }
        failure {
            echo "Pipeline FAILED. Check logs above."
        }
    }
}
