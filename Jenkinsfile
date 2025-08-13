pipeline {
    agent any

    stages {
        stage('Clone Repo') {
            steps {
                script {
                    try {
                        // Clone your public GitHub repo (no credentials needed)
                        git url: 'https://github.com/arzharch/trying_jenkins_trigger.git'

                        echo 'Flow successful'
                    } catch (err) {
                        error "Cloning failed: ${err.message}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
