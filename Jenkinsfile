pipeline {
    agent any

    environment {
        SONAR_HOME = tool "SonarScanner"
        NODEJS_HOME = tool "node23"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for frontend image')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for backend image')
    }

    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace cleanup") {
            steps {
                cleanWs()
            }
        }

        stage('Git: Code Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/manikiran7/Wanderlust-Mega-Project.git'
            }
        }

        stage("Trivy: Filesystem scan") {
            steps {
                sh '''
                echo "Running Trivy scan..."
                trivy fs --exit-code 0 --severity HIGH,CRITICAL .
                '''
            }
        }

       stage("Install dependencies (npm)") {
            steps {
                    withEnv(["PATH+NODE=${NODEJS_HOME}/bin"]) {
                    dir('frontend') {
                    sh 'npm install'
                    }
                    dir('backend') {
                    sh 'npm install'
                    }
                }
            }
        }

       stage("OWASP: Dependency check") {
            steps {
                // Use catchError to prevent breaking the pipeline
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh '''
                    echo "Running OWASP Dependency Check..."
                    /opt/dependency-check/bin/dependency-check.sh \
                      --project DP-Check \
                      --scan ./frontend --scan ./backend \
                      --out ./owasp-report \
                      --data /tmp/owasp-data \
                      --disableYarnAudit \
                      --failOnCVSS 11
                    '''
                }
            }
        }


        stage("SonarQube: Code Analysis") {
            steps {
                withSonarQubeEnv('mysonarqube') {
                    sh '''
                    ${SONAR_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=wanderlust \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage("SonarQube: Code Quality Gate") {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Exporting environment variables') {
            parallel {
                stage("Backend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }

                stage("Frontend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }

        stage("Docker: Build Images") {
            steps {
                script {
                    dir('backend') {
                        sh "docker build -t manikiran7/backend:${params.BACKEND_DOCKER_TAG} ."
                    }
                    dir('frontend') {
                        sh "docker build -t manikiran7/frontend:${params.FRONTEND_DOCKER_TAG} ."
                    }
                }
            }
        }

        stage("Docker: Push to DockerHub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push manikiran7/backend:${BACKEND_DOCKER_TAG}
                    docker push manikiran7/frontend:${FRONTEND_DOCKER_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
