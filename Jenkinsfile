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
                        sh 'mvn clean package -DskipTests'
                    } else {
                        echo 'Building only changed services...'
                        def services = env.CHANGED_SERVICES.split(',')
                        def builds = [:]

                        services.each { service ->
                            builds[service] = {
                                echo "Building service: ${service}"
                                sh "cd ${service} && mvn clean package -DskipTests"
                            }
                        }

                        parallel builds
                    }
                }
            }
        }

        stage('Docker Build & push') {
            // TODO: Check is dockerfile is exists
            // TODO: Check if /target directory is exists
            // TODO: Check if jar is exists

            steps {
                script {
                    echo "Logging into Docker Hub..."
                    sh 'echo $DOCKERHUB_PSW docker login -u $DOCKERHUB_USR --password-stdin'

                    echo 'Building Docker images for changed services...'
                    def services = env.CHANGED_SERVICES.split(',')
                    def dockerBuilds = [:]

                    services.each { service ->
                        dockerBuilds[service] = {
                            echo "Building Docker image for service: ${service} with tag ${TAG}"
                            sh "cd ${service}"

                            if (!fileExists("Dockerfile")) {
                                error "Dockerfile not found in ${service} directory."
                            }

                            echo "Dockerfile exists for service: ${service}"

                            if (!fileExists("target")) {
                                error "Target directory not found in ${service} directory."
                            }

                            if (!fileExists("target/*.jar")) {
                                error "JAR file not found in target directory of ${service}."
                            }

                            echo "Target directory and JAR file exist for service: ${service}. Proceeding with Docker build."
                            echo ""
                            
                            echo "Building Docker image for service: ${service} with tag ${TAG}"
                            sh "docker build -t ${DOCKERHUB_USR}/${service}:${TAG} ."

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