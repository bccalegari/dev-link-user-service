pipeline {
    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    agent {
        kubernetes {
            inheritFrom 'deploy-agent'
        }
    }

    parameters {
        choice(name: 'VERSION_BUMP', choices: ['patch', 'minor', 'major'], description: 'Increment type for release version')
    }

    environment {
        REGISTRY      = "registry.devlink.svc.cluster.local:5000"
        IMAGE_NAME    = "user-service"
        K8S_NAMESPACE = "devlink"
        GIT_BRANCH    = "main"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh '''
                        chmod +x mvnw
                        ./mvnw -ntp clean package -DskipTests
                    '''
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw test'
                }
            }
        }

        stage('Set Version') {
            steps {
                container('maven') {
                    script {
                        def current = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()

                        if (!current) {
                            error("Could not determine current project version")
                        }

                        def parts = current.split('\\.')
                        def major = parts[0].toInteger()
                        def minor = parts[1].toInteger()
                        def patch = parts[2].toInteger()

                        if (params.VERSION_BUMP == 'patch') patch++
                        if (params.VERSION_BUMP == 'minor') { minor++; patch = 0 }
                        if (params.VERSION_BUMP == 'major') { major++; minor = 0; patch = 0 }

                        env.RELEASE_VERSION = "${major}.${minor}.${patch}"

                        def commitHash = sh(
                            script: "git rev-parse --short HEAD",
                            returnStdout: true
                        ).trim()

                        env.IMAGE_TAG = "${env.RELEASE_VERSION}-g${commitHash}"

                        sh "mvn versions:set -DnewVersion=${env.RELEASE_VERSION} versions:commit"

                        echo "Release version: ${env.RELEASE_VERSION}"
                        echo "Image tag: ${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Generate Changelog') {
            steps {
                script {
                    def lastTag = sh(
                        script: "git describe --tags --abbrev=0 || echo ''",
                        returnStdout: true
                    ).trim()

                    def log = ""
                    if (lastTag) {
                        log = sh(
                            script: "git log ${lastTag}..HEAD --pretty=format:'- %h %B'",
                            returnStdout: true
                        ).trim()
                    } else {
                        log = sh(
                            script: "git log HEAD --pretty=format:'- %h %B'",
                            returnStdout: true
                        ).trim()
                    }

                    env.CHANGELOG = log
                    echo "Changelog:\n${env.CHANGELOG}"
                }
            }
        }

        stage('Docker Build') {
            steps {
                container('buildah') {
                    sh "buildah bud -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                container('buildah') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'devlink-registry',
                            usernameVariable: 'REGISTRY_USER',
                            passwordVariable: 'REGISTRY_PASS'
                        )
                    ]) {
                        sh '''
                        buildah login -u "$REGISTRY_USER" -p "$REGISTRY_PASS" --tls-verify=false $REGISTRY
                        buildah push --tls-verify=false $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                        '''
                    }
                }
            }
        }

        stage('Checkout IaC') {
            steps {
                dir('dev-link-iac') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: 'main']],
                        userRemoteConfigs: [[
                            url: 'https://github.com/bccalegari/dev-link-iac.git'
                        ]]
                    ])
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                container('terraform-k8s') {
                    dir('dev-link-iac/apps/user-service') {
                        script {
                            echo "Blue/Green Deployment in progress..."
                            def svcExists = sh(script: "kubectl get svc ${IMAGE_NAME} -n ${K8S_NAMESPACE}", returnStatus: true) == 0
                            def activeColor = ""
                            if (svcExists) {
                                activeColor = sh(script: "kubectl get svc ${IMAGE_NAME} -n ${K8S_NAMESPACE} -o jsonpath='{.spec.selector.color}'", returnStdout: true).trim()
                            }
                            def newColor = (!activeColor || activeColor == "green") ? "blue" : "green"
                            echo "Active color: ${activeColor ?: 'none'}"
                            echo "Deploying color: ${newColor}"

                            sh """
                                terraform init -input=false
                                terraform plan \
                                    -var="image_tag=${IMAGE_TAG}" \
                                    -var="color=${newColor}" \
                                    -var-file=${WORKSPACE}/.tfvars \
                                    -out=tfplan
                            """

                            def deploymentCreated = false
                            try {
                                input message: "Apply deployment for ${newColor}?", ok: "Deploy"
                                sh "terraform apply -auto-approve tfplan"
                                deploymentCreated = true

                                sh "kubectl rollout status deployment/${IMAGE_NAME}-${newColor} -n ${K8S_NAMESPACE} --timeout=5m"

                                def healthy = false
                                for (int i = 0; i < 12; i++) {
                                    def status = sh(script: """
                                        curl -s --connect-timeout 2 -o /dev/null \
                                        -w "%{http_code}" http://${IMAGE_NAME}.devlink.svc.cluster.local/actuator/health || true
                                    """, returnStdout: true).trim()

                                    echo "Health check attempt ${i + 1}: ${status}"
                                    if (status == "200") { healthy = true; break }
                                    sleep 5
                                }

                                if (!healthy) {
                                    error("Health check failed")
                                }

                                input message: "Promote ${newColor} to production?", ok: "Promote"
                                sh """
                                    kubectl patch svc ${IMAGE_NAME} -n ${K8S_NAMESPACE} \
                                    -p '{\"spec\":{\"selector\":{\"color\":\"${newColor}\"}}}'
                                """
                                echo "Service promoted to ${newColor}"

                            } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
                                echo "Pipeline aborted or input canceled."
                                if (deploymentCreated) {
                                    echo "Rolling back deployment ${newColor}..."
                                    sh "kubectl rollout undo deployment/${IMAGE_NAME}-${newColor} -n ${K8S_NAMESPACE}"
                                }
                                error("Pipeline aborted, rollback executed if needed.")
                            } catch (err) {
                                echo "Error during deployment: ${err}"
                                if (deploymentCreated) {
                                    echo "Rolling back deployment ${newColor}..."
                                    sh "kubectl rollout undo deployment/${IMAGE_NAME}-${newColor} -n ${K8S_NAMESPACE}"
                                }
                                error("Pipeline failed, rollback executed.")
                            }
                        }
                    }
                }
            }
        }

        stage('Release Version') {
            steps {
                container('gh') {
                    withCredentials([usernamePassword(
                        credentialsId: 'devlink-github',
                        usernameVariable: 'GITHUB_APP',
                        passwordVariable: 'GITHUB_ACCESS_TOKEN'
                    )]) {
                        script {
                            sh """
                                set -e
                                git config user.name "devlink-jenkins"
                                git config user.email "jenkins@devlink.local"

                                git add pom.xml
                                git commit -m "chore(release): ${RELEASE_VERSION}" || echo "Nothing to commit"

                                git tag -a "v${RELEASE_VERSION}" -m "Release ${RELEASE_VERSION}" || echo "Tag exists"

                                git push https://x-access-token:${GITHUB_ACCESS_TOKEN}@github.com/bccalegari/dev-link-user-service.git HEAD:${GIT_BRANCH} --tags
                            """

                            writeFile file: 'changelog.txt', text: env.CHANGELOG

                            sh '''
                                set -e

                                export GH_TOKEN=$GITHUB_ACCESS_TOKEN

                                if gh release view v$RELEASE_VERSION \
                                    --repo bccalegari/dev-link-user-service >/dev/null 2>&1; then
                                    echo "GitHub release v$RELEASE_VERSION already exists, skipping"
                                    exit 0
                                fi

                                gh release create v$RELEASE_VERSION \
                                    --repo bccalegari/dev-link-user-service \
                                    --title "Release $RELEASE_VERSION" \
                                    --notes-file changelog.txt
                            '''
                        }
                    }
                }
            }
        }

        stage('Release Monorepo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'devlink-github',
                                        usernameVariable: 'GITHUB_APP',
                                        passwordVariable: 'GITHUB_ACCESS_TOKEN')]) {
                    sh """
                        set -e

                        git clone --recurse-submodules https://${GITHUB_APP}:${GITHUB_ACCESS_TOKEN}@github.com/bccalegari/dev-link-monorepo.git dev-link-monorepo
                        cd dev-link-monorepo

                        git config user.name  "devlink-jenkins"
                        git config user.email "jenkins@devlink.local"

                        git submodule update --remote dev-link-user-service

                        if git diff --quiet --exit-code dev-link-user-service; then
                            echo "No submodule change, skipping commit and tag"
                        else
                            git add dev-link-user-service
                            git commit -m "chore: bump dev-link-user-service to ${RELEASE_VERSION}"
                            git push https://x-access-token:${GITHUB_ACCESS_TOKEN}@github.com/bccalegari/dev-link-monorepo.git HEAD:${GIT_BRANCH}
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            deleteDir()
        }
    }
}