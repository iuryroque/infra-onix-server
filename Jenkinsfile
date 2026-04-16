pipeline {
    agent any

    environment {
        COMPOSE_FILE = 'docker-compose.yml'
        DOCKER_REGISTRY = 'registry.triadevsolutions.com.br'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ?: 'latest'}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out source code..."
                    checkout scm
                }
            }
        }

        stage('Load Environment') {
            steps {
                script {
                    echo "Loading environment variables..."
                    def envFile = readFile('.env').trim()
                    envFile.split('\n').each { line ->
                        if (line && !line.startsWith('#')) {
                            def parts = line.split('=', 2)
                            if (parts.length == 2) {
                                env[parts[0].trim()] = parts[1].trim()
                            }
                        }
                    }
                }
            }
        }

        stage('Build Images') {
            steps {
                script {
                    echo "Building Docker images..."

                    // Build error-page-404
                    dir('nginx-404') {
                        sh "docker build -t error-page-404:${IMAGE_TAG} ."
                    }

                    // Build git-service if exists
                    if (fileExists('services/git-service')) {
                        dir('services/git-service') {
                            sh "docker build -t git-service:${IMAGE_TAG} ."
                        }
                    }

                    // Build orchestrator-service if exists
                    if (fileExists('services/orchestrator-service')) {
                        dir('services/orchestrator-service') {
                            sh "docker build -t orchestrator-service:${IMAGE_TAG} ."
                        }
                    }
                }
            }
        }

        stage('Validate Compose') {
            steps {
                script {
                    echo "Validating docker-compose configuration..."
                    sh "docker-compose config --quiet"
                }
            }
        }

        stage('Pull Dependencies') {
            steps {
                script {
                    echo "Pulling base images..."
                    sh "docker-compose pull db cloudflared traefik gitea registry jenkins"
                }
            }
        }

        stage('Deploy Infrastructure') {
            steps {
                script {
                    echo "Deploying infrastructure with docker-compose..."
                    sh "docker-compose up -d --remove-orphans"
                }
            }
        }

        stage('Wait for Services') {
            steps {
                script {
                    echo "Waiting for services to be healthy..."

                    // Wait for Traefik
                    waitUntil {
                        sh "docker inspect -f {{.State.Health.Status}} traefik 2>/dev/null | grep -q starting || docker inspect -f {{.State.Health.Status}} traefik 2>/dev/null | grep -q healthy || exit 1"
                        return true
                    }

                    // Wait for Gitea
                    waitUntil {
                        sh "sleep 10 && docker inspect -f {{.State.Health.Status}} gitea 2>/dev/null | grep -q healthy || exit 1"
                        return true
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying services are running..."
                    sh """
                        docker ps --filter "name=traefik" --filter "name=gitea" --filter "name=jenkins" --format "table {{.Names}}\t{{.Status}}"
                    """
                }
            }

            parallel (
                "Traefik Health": {
                    sh "curl -sf http://localhost:80/api/overview || exit 1"
                },
                "Gitea Health": {
                    sh "curl -sf http://localhost:3000/ || exit 1"
                }
            )
        }

        stage('Cleanup') {
            steps {
                script {
                    echo "Cleaning up unused images..."
                    sh "docker image prune -af --filter 'until=24h' || true"
                }
            }
        }
    }

    post {
        success {
            echo "Infrastructure deployed successfully!"
            emailext(
                subject: "SUCCESS: Infra Deployment #${env.BUILD_NUMBER}",
                body: """
                    <h3>Infra Deployment Completed Successfully</h3>
                    <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                    <p><b>Branch:</b> ${env.GIT_BRANCH}</p>
                    <p><b>Commit:</b> ${env.GIT_COMMIT}</p>
                    <p><b>Services:</b></p>
                    <pre>${sh(returnStdout: true, script: 'docker-compose ps')}</pre>
                """,
                to: env.NOTIFICATION_EMAIL ?: ''
            )
        }
        failure {
            echo "Infrastructure deployment failed!"
            emailext(
                subject: "FAILURE: Infra Deployment #${env.BUILD_NUMBER}",
                body: """
                    <h3>Infra Deployment Failed</h3>
                    <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                    <p><b>Branch:</b> ${env.GIT_BRANCH}</p>
                    <p><b>Commit:</b> ${env.GIT_COMMIT}</p>
                    <p><b>Failed Stage:</b> ${env.STAGE_NAME}</p>
                    <p><b>Error Log:</b></p>
                    <pre>${sh(returnStdout: true, script: 'docker-compose logs --tail=50')}</pre>
                """,
                to: env.NOTIFICATION_EMAIL ?: ''
            )
        }
        always {
            script {
                echo "Archiving logs..."
                sh "docker-compose logs --tail=100 > docker-compose-${env.BUILD_NUMBER}.log || true"
                archiveArtifacts artifacts: "docker-compose-${env.BUILD_NUMBER}.log", allowEmptyArchive: true
            }
        }
    }
}