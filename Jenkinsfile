import groovy.json.JsonSlurper

pipeline {
    agent any

    triggers {
        githubPush()  // Trigger build on GitHub push
    }

    environment {
        LOG_FILE = 'C:/Users/arsh.chowhan/Desktop/logs.txt'
        PROJECT_ID = 'ARSHCHOWHAN'
        TAG = 'jenkins'
        STATUS = 'design'
        API_URL = 'https://tests.api.usemango.co.uk/v1'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Fetch and Run useMango Tests') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'f722263f0311e66d', variable: 'USEMANGO_API_KEY')]) {
                        def tests = fetchTestsFromUseMango(env.API_URL, env.PROJECT_ID, env.TAG, env.STATUS, USEMANGO_API_KEY)
                        def testJobs = [:]

                        tests.each { test ->
                            testJobs["Test: ${test.Name}"] = {
                                node {
                                    dir("${env.WORKSPACE}/${test.Id}") {
                                        deleteDir()
                                        // Log start
                                        bat "echo Running Test: ${test.Name} >> \"${env.LOG_FILE}\""
                                        try {
                                            // Download .pyz
                                            bat "curl -s -L -X GET \"${env.API_URL}/tests/${test.Id}.pyz\" -H \"Authorization: APIKEY ${USEMANGO_API_KEY}\" --output ${test.Id}.pyz"
                                            // Run test
                                            bat "\"%UM_PYTHON_PATH%\" ${test.Id}.pyz"
                                            // Log success
                                            bat "echo Test ${test.Name} completed >> \"${env.LOG_FILE}\""
                                        } catch (Exception ex) {
                                            // Log failure
                                            bat "echo Test ${test.Name} failed: ${ex.getMessage()} >> \"${env.LOG_FILE}\""
                                        }
                                    }
                                }
                            }
                        }

                        parallel testJobs
                    }
                }
            }
        }

        stage('Post Execution') {
            steps {
                echo "Tests completed. Check logs at ${env.LOG_FILE}"
            }
        }
    }
}

// Helper function
def fetchTestsFromUseMango(apiUrl, projectId, tag, status, apiKey) {
    def fullUrl = "${apiUrl}/projects/${projectId}/testindex?tags=${tag}&status=${status}"
    def response = httpRequest customHeaders: [[name: 'Authorization', value: "APIKEY ${apiKey}"]],
                               url: fullUrl,
                               validResponseCodes: '200'
    def json = new JsonSlurper().parseText(response.content)
    return json.Items
}
