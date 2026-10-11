pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_ID = 'taherlammouchi'
        MOVIE_IMAGE = 'movie-service'
        CAST_IMAGE = 'cast-service'
        DOCKER_TAG = "v${BUILD_NUMBER}"
        MOVIE_CHART = 'charts'
        CAST_CHART = 'charts'
    }

    stages {
        stage('Verification') {
            steps {
                sh '''
                    set -eu
                    docker --version
                    kubectl version --client
                    helm version
                    test -f movie-service/Dockerfile
                    test -f cast-service/Dockerfile
                    helm lint charts/
                '''
            }
        }

        stage('Tests') {
            steps {
                sh '''
                    set -eu

                    echo "=== Vérification des fichiers ==="
                    test -f movie-service/Dockerfile
                    test -f cast-service/Dockerfile

                    echo "=== Validation du chart Helm ==="
                    helm lint charts/

                    echo "=== Vérification du rendu Kubernetes ==="
                    helm template movie-service charts/ -n dev > /tmp/movie-service.yaml
                    helm template cast-service charts/ -n dev > /tmp/cast-service.yaml

                    echo "Tests de validation terminés."
                '''
            }
        }

        stage('Build images') {
            steps {
                sh '''
                    set -eu
                    docker build -t "$DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG" ./movie-service
                    docker build -t "$DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG" ./cast-service
                '''
            }
        }

        stage('Push DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$REGISTRY_PASSWORD" | docker login \
                            -u "$REGISTRY_USER" --password-stdin
                        docker push "$DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG"
                        docker push "$DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            when { branch 'dev' }
            steps { deployToNamespace('dev') }
        }

        stage('Tests CI DEV') {
            when {
                branch 'dev'
            }
            steps {
                withCredentials([
                    file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl get pods -n dev
                        kubectl rollout status deployment/movie-service \
                            -n dev --timeout=180s
                        kubectl rollout status deployment/cast-service \
                            -n dev --timeout=180s
                    '''
                }
            }
        }

        stage('Deploy QA') {
            when { branch 'qa' }
            steps { deployToNamespace('qa') }
        }

        stage('Deploy STAGING') {
            when { branch 'staging' }
            steps { deployToNamespace('staging') }
        }

        stage('Approval PROD') {
            when { branch 'main' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Autoriser le déploiement en production ?',
                        ok: 'Déployer'
                    )
                }
            }
        }

        stage('Deploy PROD') {
            when { branch 'main' }
            steps { deployToNamespace('prod') }
        }
    }

    post {
        success {
            echo 'Pipeline terminé avec succès.'
        }
        failure {
            echo 'Échec du pipeline : vérifier les logs.'
        }
    }
}

def deployToNamespace(String namespace) {
    withCredentials([
        file(credentialsId: 'conf', variable: 'KUBECONFIG_FILE')
    ]) {
        withEnv(["TARGET_NAMESPACE=${namespace}"]) {
            sh '''
                set -eu
                export KUBECONFIG="$KUBECONFIG_FILE"

                kubectl get namespace "$TARGET_NAMESPACE"

                helm upgrade --install movie-service "$MOVIE_CHART" \
                    -n "$TARGET_NAMESPACE" \
                    --set image.repository="$DOCKER_ID/$MOVIE_IMAGE" \
                    --set image.tag="$DOCKER_TAG" \
                    --wait --timeout 180s

                helm upgrade --install cast-service "$CAST_CHART" \
                    -n "$TARGET_NAMESPACE" \
                    --set image.repository="$DOCKER_ID/$CAST_IMAGE" \
                    --set image.tag="$DOCKER_TAG" \
                    --wait --timeout 180s
            '''
        }
    }
}
