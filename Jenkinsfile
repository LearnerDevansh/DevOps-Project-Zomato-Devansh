def call(Map config = [:]) {
    pipeline {
        parameters {
            choice(
        name: 'ENV',
        choices: ['dev', 'stage', 'prod'],
        description: 'Select deployment environment'
    )
        }
        agent any

        tools {
            jdk 'jdk17'
            nodejs 'node23'
        }

        environment {
            SCANNER_HOME = tool 'sonar-scanner'
            IMAGE_NAME = "${env.BUILD_NUMBER}"
            DOCKER_USER = "Devansh21"
            IMAGE_TAG = "${env.BUILD_NUMBER}"
        }

        stages {
            stage('Pipeline Mode') {
                steps {
                    script {
                        env.PIPELINE_TYPE =
                env.JOB_NAME.contains('cd') ? 'CD' : 'CI'

                        echo "Running ${env.PIPELINE_TYPE} pipeline"
                    }
                }
            }
            stage('Clean Workspace') {
                steps {
                    cleanWs()
                }
            }

            stage('Init Config') {
                steps {
                    script {
                        env.DOCKER_USER = config.dockerUser ?: 'Devansh21'
                    }
                }
            }

            stage('Prepare Environment') {
                steps {
                    script {
                        env.NAMESPACE = "${env.IMAGE_NAME}-${params.ENV}"
                        echo "Namespace: ${env.NAMESPACE}"
                    }
                }
            }

            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('SonarQube Analysis') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    withSonarQubeEnv('sonar-server') {
                        sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato
                        """
                    }
                }
            }

            stage('Quality Gate') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    waitForQualityGate abortPipeline: True,
                    credentialsId: 'Sonar-token'
                }
            }

            stage('Install Dependencies') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    sh 'npm install'
                }
            }

            stage('OWASP Scan') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    dependencyCheck additionalArguments:
                    '--scan ./ --disableYarnAudit --disableNodeAudit -n',
                    odcInstallation: 'DP-Check'

                    dependencyCheck(
                        additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit -n',
                        odcInstallation: 'DP-Check'
                    )
                }
            }

            stage('Trivy Scan') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    sh 'trivy fs . > trivy.txt'
                }
            }

            stage('Build Docker Image') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
                }
            }

            stage('Push Docker Image') {
                when {
                    expression { env.PIPELINE_TYPE == 'CI' }
                }
                steps {
                    withDockerRegistry(credentialsId: 'docker',
                    url: 'https://index.docker.io/v1/') {
                        sh """
                        docker tag ${IMAGE_NAME} ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }

            stage('Deploy with Helm') {
                when {
                    expression { env.PIPELINE_TYPE == 'CD' }
                }
                steps {
                    script {
                        def port = params.ENV == 'prod' ? 80 :
                        params.ENV == 'stage' ? 4000 : 3000

                        withKubeConfig(credentialsId: 'kubeconfig') {
                            sh """
                            helm upgrade --install ${IMAGE_NAME} ./helm \
                            --namespace ${params.ENV} \
                            --create-namespace \
                            --set image.repository=${DOCKER_USER}/${IMAGE_NAME} \
                            --set image.tag=${IMAGE_TAG} \
                            --set service.port=${port}
                          """
                        }
                    }
                }
            }
        }

        post {
            success {
                echo '✅ Deployment Successful'
            }
            failure {
                echo '❌ Deployment Failed'
            }
        }
    }
}

