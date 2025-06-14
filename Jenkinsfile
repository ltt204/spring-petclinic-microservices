pipeline {
    agent any

    environment {
        // Define any environment variables here
    }

    stage {
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
                    def changedFiles = sh(script: "git diff --name-only origin/main HEAD~1", returnStdout: true).trim().split('\n').findAll { it.trim() }
                    
                    def changedServices=changedFiles
                        .findAll {it.startsWith("spring-petclinic-") }
                        .collect { it.split('/')[0] }
                        .unique()
                    
                   if (services.isEmpty()) {
                        error "No services changed in the last commit."
                        currentBuild.result = 'TRUE'
                    } else {
                        echo "Changed services: ${changedServices.join(', ')}"
                        env.CHANGED_SERVICES = changedServices.join(',')
                    }
                }
            }
        }

        stage('Build & Test') {
            // Build the project
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',')
                    def builds = [:]

                    services.each { service ->
                        builds[service] = {
                            echo "Building service: ${service}"
                            sh "cd ${service} && ./mvnw clean package -DskipTests"
                        }
                    }

                    parallel builds
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