pipeline {
    agent any

    environment {
        // Define any environment variables here
        BUILD_ALL = 'false'
        DOCKERHUB = credentials('dockerhub') // Replace with your actual credentials ID
        TAG = "LATEST"
        FAILED_STAGES = ''
    }

    stages {
        stage('Checkout') {
            // Checkout the code from the repository
            steps {
                checkout scm
                sh "git fetch origin main"
            }
        }

        stage('Detects change') {
            // Detect changes in the repository
            steps {
                script {
                    echo 'Detecting changes in the repository...'
                    def changedFiles = sh(script: "git diff --name-only origin/main HEAD", returnStdout: true).trim().split('\n').findAll { it.trim() }
                    
                    def changedServices=changedFiles
                        .findAll {it.startsWith("spring-petclinic-") }
                        .collect {it.split('/')[0] }
                        .unique()
                    
                    if (changedServices.isEmpty()) {
                        echo "No services changed in the last commit."
                        BUILD_ALL = 'true'
                    } else {
                        echo "Changed services: ${changedServices.join(', ')}"
                        def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        TAG = commitId
                        env.CHANGED_SERVICES = changedServices.join(',')
                    }

                }
            }
        }

        stage('Build & Test') {
            // Build the project
            steps {
                script {
                    if (BUILD_ALL == 'true') {
                        echo 'Building all services...'
                        sh './mvnw clean package -DskipTests'
                    } else {
                        echo 'Building only changed services...'
                        def services = env.CHANGED_SERVICES.split(',')
                        def builds = [:]

                        services.each { service ->
                            builds[service] = {
                                echo "Building service: ${service}"
                                sh "cd ${service} && ../mvnw clean package -DskipTests"
                            }
                        }

                        parallel builds
                    }
                }
            }
        }

        stage('Docker Build & push') {
            steps {
                script {
                    echo "Logging into Docker Hub..."
                    sh "echo $DOCKERHUB_PWD docker login -u $DOCKERHUB_USR --password-stdin"

                    echo 'Building Docker images for changed services...'
                    def services = env.CHANGED_SERVICES.split(',')
                    def dockerBuilds = [:]

                    services.each { service ->
                        dockerBuilds[service] = {
                            echo "Building Docker image for service: ${service} with tag ${TAG}"
                            sh "cd ${service} && docker build -t ${DOCKERHUB_USR}/${service}:${TAG} ."
                            echo "Pushing Docker image for service: ${service} with tag ${TAG}"
                            sh "docker push ${DOCKERHUB_USR}/${service}:${TAG}"
                            echo "Docker image for service ${service} pushed successfully."
                        }
                    }

                    parallel dockerBuilds
                }
            }
        }
    }

    post {
        always {
            // Archive the artifacts
            archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
        }

        success {
            echo 'Build and tests completed successfully.'
        }

        failure {
            echo 'Build or tests failed.'
        }

        cleanup {
            echo 'Cleaning up...'
            // Perform any necessary cleanup actions
            cleanWs()
        }
    }
}