pipeline {
    environment {
        DOCKER_ID = "neod13f"
        DOCKER_IMAGE = "exam-app"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    agent any

    stages {
        stage('Docker Build & Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
                        echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
                        docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Déploiements (dev, qa, staging)') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    ['dev', 'qa', 'staging'].each { envName ->
                        sh """
                            mkdir -p .kube
                            cat \$KUBECONFIG > .kube/config
                            cp charts/values.yaml values.yml
                            sed -i "s+repository.*+repository: ${DOCKER_ID}/${DOCKER_IMAGE}+g" values.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                            helm upgrade --install app ./charts --values=values.yml --namespace ${envName}
                        """
                    }
                }
            }
        }

        stage('Déploiement en prod') {
            when {
                branch 'main'
            }
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Souhaitez-vous déployer en production ?', ok: 'Oui'
                }
                script {
                    sh '''
                        mkdir -p .kube
                        cat $KUBECONFIG > .kube/config
                        cp charts/values.yaml values.yml
                        sed -i "s+repository.*+repository: ${DOCKER_ID}/${DOCKER_IMAGE}+g" values.yml
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app ./charts --values=values.yml --namespace prod
                    '''
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
