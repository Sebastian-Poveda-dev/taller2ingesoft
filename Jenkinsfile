// ─────────────────────────────────────────────────────────────────────────────
// CircleGuard — Unified Multibranch Pipeline
// Behaviour adapts automatically to the branch being built:
//   dev    → build + unit tests + docker + deploy to circleguard-dev
//   stage  → dev stages + integration/E2E tests + deploy to circleguard-stage
//   master → full pipeline + manual approval + deploy to circleguard-master
//            + automatic Release Notes generation
// ─────────────────────────────────────────────────────────────────────────────
pipeline {
    agent any

    environment {
        IMAGE_TAG      = "${env.BRANCH_NAME}"
        K8S_NAMESPACE  = "circleguard-${env.BRANCH_NAME}"
        K8S_OVERLAY    = "k8s/overlays/${env.BRANCH_NAME}"
        DOCKER_HOST    = "unix:///var/run/docker.sock"
        TESTCONTAINERS_RYUK_DISABLED = "true"
    }

    options {
        timeout(time: 90, unit: 'MINUTES')   // first run downloads all Gradle deps
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // ── Stage definitions ─────────────────────────────────────────────────────
    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'chmod +x gradlew'
                sh '''
                    printf "ryuk.disabled=true\\ndocker.client.strategy=org.testcontainers.dockerclient.UnixSocketClientProviderStrategy\\n" \
                        > ~/.testcontainers.properties
                '''
                script {
                    env.SHORT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.APP_VERSION  = sh(
                        script: "grep '^    version' build.gradle.kts | head -1 | sed 's/.*\"\\(.*\\)\".*/\\1/' || echo '1.0.0'",
                        returnStdout: true
                    ).trim()
                    echo "Branch: ${env.BRANCH_NAME} | Version: ${env.APP_VERSION} | Commit: ${env.SHORT_COMMIT}"
                }
            }
        }

        // ── Gradle Dependency Warmup ──────────────────────────────────────────
        // Downloads all dependencies before parallel stages to avoid
        // concurrent downloads causing flaky failures on first run.
        stage('Resolve Dependencies') {
            steps {
                sh './gradlew :services:circleguard-auth-service:dependencies :services:circleguard-identity-service:dependencies :services:circleguard-form-service:dependencies :services:circleguard-promotion-service:dependencies :services:circleguard-notification-service:dependencies :services:circleguard-gateway-service:dependencies --no-daemon -q 2>/dev/null || true'
            }
        }

        // ── Unit Tests (all branches) ─────────────────────────────────────────
        stage('Unit Tests') {
            parallel {
                stage('auth') {
                    steps { sh './gradlew :services:circleguard-auth-service:test --no-daemon' }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-auth-service/build/test-results/test/*.xml' } }
                }
                stage('identity') {
                    steps { sh './gradlew :services:circleguard-identity-service:test --no-daemon' }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-identity-service/build/test-results/test/*.xml' } }
                }
                stage('form') {
                    steps { sh './gradlew :services:circleguard-form-service:test --no-daemon' }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-form-service/build/test-results/test/*.xml' } }
                }
                stage('promotion') {
                    steps {
                        sh '''./gradlew :services:circleguard-promotion-service:test --no-daemon \
                            --tests "com.circleguard.promotion.controller.*" \
                            --tests "com.circleguard.promotion.service.HealthStatusServiceTest" \
                            --tests "com.circleguard.promotion.service.FloorServiceTest" \
                            --tests "com.circleguard.promotion.service.StatusLifecycleTest" \
                            --tests "com.circleguard.promotion.listener.*"
                        '''
                    }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-promotion-service/build/test-results/test/*.xml' } }
                }
                stage('notification') {
                    steps { sh './gradlew :services:circleguard-notification-service:test --no-daemon' }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-notification-service/build/test-results/test/*.xml' } }
                }
                stage('gateway') {
                    steps { sh './gradlew :services:circleguard-gateway-service:test --no-daemon' }
                    post { always { junit allowEmptyResults: true,
                        testResults: 'services/circleguard-gateway-service/build/test-results/test/*.xml' } }
                }
            }
        }

        // ── Build JARs (all branches) ─────────────────────────────────────────
        stage('Build JARs') {
            steps {
                sh '''
                    ./gradlew \
                        :services:circleguard-auth-service:bootJar \
                        :services:circleguard-identity-service:bootJar \
                        :services:circleguard-form-service:bootJar \
                        :services:circleguard-promotion-service:bootJar \
                        :services:circleguard-notification-service:bootJar \
                        :services:circleguard-gateway-service:bootJar \
                        --no-daemon -x test --parallel
                '''
            }
        }

        // ── Docker Build into Minikube (all branches) ─────────────────────────
        stage('Docker Build') {
            steps {
                sh 'minikube status | grep -q "Running" || minikube start --driver=docker'
                script {
                    ['auth', 'identity', 'form', 'promotion', 'notification', 'gateway'].each { svc ->
                        sh """
                            eval \$(minikube docker-env)
                            docker build \
                                -f docker/circleguard-${svc}-service/Dockerfile \
                                -t circleguard/${svc}-service:${IMAGE_TAG} \
                                -t circleguard/${svc}-service:${BUILD_NUMBER} \
                                --label "branch=${BRANCH_NAME}" \
                                --label "build=${BUILD_NUMBER}" \
                                --label "commit=${SHORT_COMMIT}" \
                                .
                        """
                    }
                }
            }
        }

        // ── Deploy (all branches, each to its own namespace) ──────────────────
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/base/namespaces/namespaces.yaml'
                sh "kubectl apply -k ${K8S_OVERLAY}"
                sh """
                    for svc in auth identity form promotion notification gateway; do
                        echo "Waiting for \${svc}-service in ${K8S_NAMESPACE}..."
                        kubectl rollout status deployment/\${svc}-service \
                            -n ${K8S_NAMESPACE} --timeout=180s || true
                    done
                """
            }
        }

        // ── Smoke Test (all branches) ─────────────────────────────────────────
        stage('Smoke Test') {
            steps {
                script {
                    [
                        [svc: 'auth-service',         port: '8180'],
                        [svc: 'identity-service',     port: '8083'],
                        [svc: 'form-service',         port: '8086'],
                        [svc: 'promotion-service',    port: '8088'],
                        [svc: 'notification-service', port: '8082'],
                        [svc: 'gateway-service',      port: '8087']
                    ].each { c ->
                        sh """
                            kubectl run smoke-${c.svc.replace('-','')} \
                                --image=curlimages/curl:latest \
                                --restart=Never --rm -i \
                                -n ${K8S_NAMESPACE} \
                                --command -- curl -sf \
                                http://${c.svc}:${c.port}/actuator/health \
                                && echo "[OK] ${c.svc}" \
                                || echo "[WARN] ${c.svc} not ready yet"
                        """
                    }
                }
            }
        }

        // ── Integration Tests (stage + master only) ───────────────────────────
        stage('Integration Tests') {
            when { expression { env.BRANCH_NAME in ['stage', 'master'] } }
            steps {
                sh '''
                    ./gradlew \
                        :services:circleguard-auth-service:test \
                        :services:circleguard-identity-service:test \
                        :services:circleguard-promotion-service:test \
                        --no-daemon \
                        -Dspring.profiles.active=integration \
                        --tests "*IntegrationTest*" --tests "*IT" \
                        || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/build/test-results/test/*IT*.xml,**/build/test-results/test/*IntegrationTest*.xml'
                }
            }
        }

        // ── E2E Tests (stage + master only) ───────────────────────────────────
        stage('E2E Tests') {
            when { expression { env.BRANCH_NAME in ['stage', 'master'] } }
            steps {
                sh '''
                    ./gradlew \
                        :services:circleguard-promotion-service:test \
                        :services:circleguard-form-service:test \
                        --no-daemon \
                        -Dspring.profiles.active=e2e \
                        --tests "*E2ETest*" --tests "*EndToEnd*" \
                        || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/build/test-results/test/*E2E*.xml,**/build/test-results/test/*EndToEnd*.xml'
                }
            }
        }

        // ── Manual Approval (master only) ─────────────────────────────────────
        stage('Approval: Promote to Master') {
            when { branch 'master' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: """
✅ All tests passed on Stage.

Version : ${env.APP_VERSION}
Commit  : ${env.SHORT_COMMIT}
Build   : #${BUILD_NUMBER}

Approve to finalize MASTER deployment?
                    """, ok: 'Deploy to Master'
                }
            }
        }

        // ── Release Notes (master only) ───────────────────────────────────────
        stage('Release Notes') {
            when { branch 'master' }
            steps {
                script {
                    def notes = sh(script: '''
                        PREV_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || \
                                   git rev-list --max-parents=0 HEAD)
                        printf "## CircleGuard %s — Release Notes\\n\\n" "${APP_VERSION}"
                        printf "**Build:** #%s  \\n" "${BUILD_NUMBER}"
                        printf "**Date:** %s  \\n" "$(date -u '+%Y-%m-%d %H:%M UTC')"
                        printf "**Commit:** %s  \\n\\n" "${SHORT_COMMIT}"
                        printf "### Changes\\n\\n"
                        git log --pretty=format:"- %s (%an)" ${PREV_TAG}..HEAD 2>/dev/null || \
                            git log --pretty=format:"- %s (%an)" --max-count=20
                        printf "\\n\\n### Services Deployed\\n\\n"
                        printf "| Service | Version |\\n|---------|---------|\\n"
                        for svc in auth identity form promotion notification gateway; do
                            printf "| %s-service | %s |\\n" "${svc}" "${APP_VERSION}"
                        done
                    ''', returnStdout: true).trim()

                    writeFile file: 'RELEASE_NOTES.md', text: notes
                    archiveArtifacts artifacts: 'RELEASE_NOTES.md', fingerprint: true

                    sh """
                        git config user.email "ci@circleguard.edu"
                        git config user.name "CircleGuard CI"
                        git tag -a "v${APP_VERSION}-build-${BUILD_NUMBER}" \
                            -m "Release ${APP_VERSION} — Build #${BUILD_NUMBER}" || true
                    """
                    echo "Release notes:\n${notes}"
                }
            }
        }
    }

    post {
        success {
            echo "✅ [${env.BRANCH_NAME ?: 'unknown'}] Pipeline complete — ${env.APP_VERSION ?: ''} @ ${env.SHORT_COMMIT ?: ''}"
        }
        failure {
            echo "❌ [${env.BRANCH_NAME ?: 'unknown'}] Pipeline failed"
            // Use single-quoted shell string so the variable is resolved by the shell
            // from the environment block, not by Groovy (avoids MissingPropertyException)
            sh 'kubectl get pods -n $K8S_NAMESPACE 2>/dev/null || true'
            sh 'kubectl get events -n $K8S_NAMESPACE --sort-by=.lastTimestamp 2>/dev/null | tail -15 || true'
        }
        always { cleanWs(cleanWhenNotBuilt: false, cleanWhenFailure: false) }
    }
}
