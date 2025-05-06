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

	stage('Debug : Afficher la branche') {
	    steps {
	        echo "env.BRANCH_NAME = ${env.BRANCH_NAME}"
		echo "env.GIT_BRANCH = ${env.GIT_BRANCH}"
	        sh 'echo BRANCHE SHELL = $GIT_BRANCH'
	    }
	}

        stage('Déploiement (dev, qa, staging)') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    def namespaces = ['dev', 'qa', 'staging']
                    def services = ['movie-service', 'cast-service']
                    def basePort = 30007
                    def portCounter = 0

                    namespaces.each { ns ->
                        services.each { svc ->
                            def imageName = svc
                            def releaseName = svc.contains('movie') ? 'movie' : 'cast'
                            def currentPort = basePort + portCounter
                            portCounter += 1

                            // Création du fichier de configuration kubeconfig
                            // Déploiement du service avec image, tag et port dynamiquement injectés
                            sh """
                                mkdir -p .kube
                                cat \$KUBECONFIG > .kube/config

                                helm upgrade --install ${releaseName} charts \\
                                  --set image.repository=${DOCKER_ID}/${imageName} \\
                                  --set image.tag=${DOCKER_TAG} \\
                                  --set service.nodePort=${currentPort} \\
                                  --namespace ${ns}
                            """
                        }
                    }
                }
            }
        }

        stage('Déploiement en prod') {
            when {
                expression { return env.GIT_BRANCH == 'origin/master' }
            }
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Souhaitez-vous déployer en production ?', ok: 'Oui'
                }
                script {
                    def services = ['movie-service', 'cast-service']
                    def basePort = 30013
                    def portCounter = 0

                    services.each { svc ->
                        def imageName = svc
                        def releaseName = svc.contains('movie') ? 'movie' : 'cast'
                        def currentPort = basePort + portCounter
                        portCounter += 1

                        // Déploiement en namespace prod avec ports séparés pour éviter les conflits
                        sh """
                            mkdir -p .kube
                            cat \$KUBECONFIG > .kube/config

                            helm upgrade --install ${releaseName} charts \\
                              --set image.repository=${DOCKER_ID}/${imageName} \\
                              --set image.tag=${DOCKER_TAG} \\
                              --set service.nodePort=${currentPort} \\
                              --namespace prod
                        """
                    }
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
