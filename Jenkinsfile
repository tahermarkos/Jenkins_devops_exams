pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        timestamps()
    }

    environment {
        DOCKER_ID = 'taherlammouchi'
        MOVIE_IMAGE = 'movie-service'
        CAST_IMAGE = 'cast-service'
        DOCKER_TAG = "v${BUILD_NUMBER}"

        // A adapter selon le contenu réel de charts/
        MOVIE_CHART = 'charts'
        CAST_CHART = 'charts'
    }

    stages {
        stage('Verification') {
            steps {
                sh '''
                    set -eu
                    git --version
                    docker --version
                    helm version
                    test -f movie-service/Dockerfile
                    test -f cast-service/Dockerfile
                '''
            }
        }

        stage('Build images') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        -t "$DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG" \
                        ./movie-service

                    docker build \
                        -t "$DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG" \
                        ./cast-service
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
                        set -eu
                        set +x

                        echo "$REGISTRY_PASSWORD" |
                            docker login \
                                --username "$REGISTRY_USER" \
                                --password-stdin

                        docker push "$DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG"
                        docker push "$DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            when {
                branch 'dev'
            }
            steps {
                deployToNamespace('dev')
            }
        }

        stage('Deploy QA') {
            when {
                branch 'qa'
            }
            steps {
                deployToNamespace('qa')
            }
        }

        stage('Deploy STAGING') {
            when {
                branch 'staging'
            }
            steps {
                deployToNamespace('staging')
            }
        }

        stage('Approval PROD') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Autoriser le deploiement en production ?',
                        ok: 'Deployer en production'
                    )
                }
            }
        }

        stage('Deploy PROD') {
            when {
                branch 'main'
            }
            steps {
                deployToNamespace('prod')
            }
        }
    }
}

// Fonction commune aux quatre environnements.
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
                    --namespace "$TARGET_NAMESPACE" \
                    --set image.repository="$DOCKER_ID/$MOVIE_IMAGE" \
                    --set image.tag="$DOCKER_TAG" \
                    --wait --timeout 180s

                helm upgrade --install cast-service "$CAST_CHART" \
                    --namespace "$TARGET_NAMESPACE" \
                    --set image.repository="$DOCKER_ID/$CAST_IMAGE" \
                    --set image.tag="$DOCKER_TAG" \
                    --wait --timeout 180s

                kubectl get pods -n "$TARGET_NAMESPACE"
            '''
        }
    }
}
