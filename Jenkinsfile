pipeline {
    agent any

    stages {
        stage('Clone Repo') {
            steps {
                echo 'Cloning public repo...'
                // Repo is auto-cloned by Jenkins when using "Pipeline script from SCM"
            }
        }

        stage('Install & Run Tests') {
            steps {
                echo 'Running tests...'
                // Replace the following line with your actual test command
                sh './run_tests.sh'
            }
        }
    }

    post {
        success {
            echo '✅ Flow successful!'
        }
        failure {
            echo '❌ Flow failed.'
        }
        always {
            cleanWs() // Clean up workspace
        }
    }
}
