pipeline {
    environment {
        DOCKER_ID = "neod13f"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    agent any

    stages {
        stage('Build & Push Docker Images') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    def services = ['movie-service', 'cast-service']
                    services.each { svc ->
                        def image = "$DOCKER_ID/${svc}:${DOCKER_TAG}"
                        sh """
                            docker build -t ${image} ./${svc}
                            echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
                            docker push ${image}
                        """
                    }
                }
            }
        }

        stage('Déploiement (dev, qa, staging)') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    def namespaces = ['dev', 'qa', 'staging']
                    namespaces.each { ns ->
                        // Si tu veux modifier les valeurs pour chaque service, ajoute ici
                        sh """
                            mkdir -p .kube
                            cat \$KUBECONFIG > .kube/config

                            cp charts/values.yaml values.yml
                            sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                            sed -i "s+repository:.*+repository: ${DOCKER_ID}/movie-service+g" values.yml
                            helm upgrade --install movie charts --values=values.yml --namespace ${ns}

                            sed -i "s+repository:.*+repository: ${DOCKER_ID}/cast-service+g" values.yml
                            helm upgrade --install cast charts --values=values.yml --namespace ${ns}
                        """
                    }
                }
            }
        }

        stage('Déploiement en prod') {
            when {
                branch 'master'
            }
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Déployer en production ?'
                }
                script {
                    sh """
                        mkdir -p .kube
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                        sed -i "s+repository:.*+repository: ${DOCKER_ID}/movie-service+g" values.yml
                        helm upgrade --install movie charts --values=values.yml --namespace prod

                        sed -i "s+repository:.*+repository: ${DOCKER_ID}/cast-service+g" values.yml
                        helm upgrade --install cast charts --values=values.yml --namespace prod
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "Le pipeline a échoué."
        }
    }
}
