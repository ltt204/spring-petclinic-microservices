pipeline {
    agent any

    environment {
        // Define any environment variables here
        // DOCKERHUB = credentials('dockerhub') // Replace with your actual credentials ID
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
                    def base = sh(script: "git merge-base origin/main HEAD", returnStdout:true).trim()
                    env.TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.CHANGED_SERVICES = sh(script: "git diff --name-only ${base} HEAD | grep '^spring-petclinic-' | cut -d/ -f1 | sort -u | paste -sd , -", returnStdout: true).trim()
                    echo "Changed services: ${env.CHANGED_SERVICES}"
                }
            }
        }

        // stage ('Build ')

        stage('Build & Test') {
            // Build the project
            steps {
                script {
                    if (env.CHANGED_SERVICES.isEmpty()) {
                        echo 'Building all services...'
                        sh 'mvn clean package -DskipTests'
                    } else {
                        echo 'Building only changed services...'
                        def services = env.CHANGED_SERVICES.split(',')
                        def builds = [:]

                        services.each { service ->
                            builds[service] = {
                                echo "Building service: ${service}"
                                sh "cd ${service} && mvn -B -T1C clean package -DskipTests"
                            }
                        }
                        
                        builds.failFast = true // Ensure that the build fails fast if any service fails
                        parallel builds
                    }
                }
            }
        }

        stage('Docker Build & push') {
            steps {
                script {
                    if (env.CHANGED_SERVICES.isEmpty()) {
                        echo 'No services changed, skipping Docker build and push.'
                        return
                    }

                    sh "./create-dockerfiles.sh"

                    echo "Logging into Docker Hub..."
                    withCredentials([usernamePassword(
                            credentialsId: 'dockerhub', 
                            usernameVariable: 'DOCKERHUB_USR', 
                            passwordVariable: 'DOCKERHUB_PSW')
                    ]) {
                        env.DOCKERHUB_USR = "${DOCKERHUB_USR}"
                        
                        sh 'echo $DOCKERHUB_PSW | docker login -u $DOCKERHUB_USR --password-stdin'

                        echo 'Building Docker images for changed services...'
                        def services = env.CHANGED_SERVICES.split(',')
                        def dockerBuilds = [:]

                        services.each { service ->
                            dockerBuilds[service] = {
                                dir (service) {
                                    echo "Building Docker image for service: ${service} with tag ${TAG}"

                                    if (!fileExists("Dockerfile")) {
                                        error "Dockerfile not found in ${service} directory."
                                    }
                                    echo "Dockerfile exists for service: ${service}"

                                    echo "Checking for target directory in ${service}..."
                                    if (!fileExists("target")) {
                                        error "Target directory not found in ${service} directory."
                                    }
                                    echo "Target directory exists for service: ${service}"

                                    echo "Checking for JAR file in target directory of ${service}..."
                                    def jarExists = sh(
                                            script: 'ls target/*.jar 2>/dev/null | wc -l',
                                            returnStdout: true
                                        ).trim() != '0'
                                    if (!jarExists) {
                                        error "JAR file not found in target directory of ${service}."
                                    }
                                    echo "JAR file exists in target directory of ${service} directory."

                                    echo "Target directory and JAR file exist for service: ${service}. Proceeding with Docker build."
                                    echo ""

                                    echo "Building Docker image for service: ${service} with tag ${TAG}"
                                    sh "docker build -t ${env.DOCKERHUB_USR}/${service}:${TAG} -t ${env.DOCKERHUB_USR}/${service}:latest ."

                                    // lock(resource: 'staging-server', inversePrecedence: true) {
                                    echo "Pushing Docker image for service: ${service} to Docker Hub"
                                    sh "docker push ${env.DOCKERHUB_USR}/${service}:${TAG}"
                                    sh "docker push ${env.DOCKERHUB_USR}/${service}:latest"
                                    // }

                                    echo "Docker image for service ${service} pushed successfully."
                                }
                            }
                        }

                        dockerBuilds.failFast = true // Ensure that the build fails fast if any service fails
                        parallel dockerBuilds
                    }
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
            sh "docker image prune -af"
            cleanWs()
        }
    }
}