import groovy.json.JsonSlurper
import java.net.URL
import java.net.HttpURLConnection
import java.net.URLEncoder
import java.util.NoSuchElementException

def getRunId() {
    if (fileExists("multidataset_run.log")) {
        return powershell(returnStdout: true, script: 'Write-Output (Get-Content .\\multidataset_run.log | select -First 1 | Select-String -Pattern \'.*\\"RunId\\": \\"([^\\"]*)\\"\').Matches.Groups[1].Value')
    }
    if (fileExists("run.log")) {
        return powershell(returnStdout: true, script: 'Write-Output (Get-Content .\\run.log | select -First 1 | Select-String -Pattern \'.*\\"RunId\\": \\"([^\\"]*)\\"\').Matches.Groups[1].Value')
    }
    return null
}

def getTests(String baseUrl) {
    String cursor = ""
    def tests = []
    def jsonSlurper = new JsonSlurper()
    echo "Retrieved the following tests from project ${params['Project ID']} with the tags ${params['Tags']} and status ${params['Status']}"
    while(true) {
        URL url = new URL("${baseUrl}/projects/${params['Project ID']}/testindex?tags=${params['Tags']}&status=${params['Status']}&cursor=${cursor}")
        def testPage = getRequest(url, "Testindex")
        testPage.Items.each{ test ->
            def scenarios = getScenarios(baseUrl, test.Id)
            tests << [Id: test.Id, Name: test.Name, Scenarios: scenarios]
        }
        if (!testPage.Info.HasNext){
            break
        }
        cursor = testPage.Info.Next
    }
    return tests
}

def getScenarios(String baseUrl, String testId){
    def scenarios = [[Id: "0", Name: "Default"]]
    def runWithDataset = isRunWithDatasetOptionSelected()
    if (runWithDataset) {
        def isScenarioPresentForTest = scenariosPresent(baseUrl, testId)
        if (isScenarioPresentForTest) {
            URL url = new URL("${baseUrl}/projects/${params['Project ID']}/tests/${testId}/scenarios")
            def scenarioPage = getRequest(url, "Scenarios")
            if (scenarioPage != null) {
                scenarioPage.each { scenario ->
                    scenarios << [Id: scenario.Id, Name: scenario.Name]
                }
            }
            return scenarios;
        }
    }
    return null
}

def addQueryParameterToUrl(String path, Map<String, Object> queryParams) {
    if (queryParams.isEmpty()) {
        return new URL(path)
    }
    path = path + "?"
    for (final def keyValue in queryParams.entrySet()) {
        def query = keyValue.getKey()
        def queryValue = keyValue.getValue()
        if (queryValue instanceof ArrayList) {
            queryValue.each { value -> 
                path += "${query}=${URLEncoder.encode(value as String, 'UTF-8')}&"
            }
        } else {
            path += "${query}=${URLEncoder.encode(queryValue as String, "UTF-8")}&"
        }
    }
    return new URL(path.substring(0, path.length() - 1))
}

boolean scenariosPresent(String baseUrl, String testId) {
    URL url = new URL("${baseUrl}/projects/${params['Project ID']}/tests/${testId}")
    def scenarioPage = getRequest(url, "Dataset")
    return scenarioPage["Parameters"].size() >= 1
}

boolean isRunWithDatasetOptionSelected() {
    def value = params['Run with datasets']
    if (value != null) {
        return value
    }
    // backward compatibility
    value = params['Run with scenarios']
    if (value != null) {
        return value
    }
}

def getEnvIdAndName(String baseUrl) {
    echo "Retrieving environment"
    if (params['Environment'] == null || params['Environment'].toString().isEmpty()) {
        def (envId, envName) = getDefaultEnv(baseUrl)
        echo "Using the current default environment '${envName}'"
        return [envId, envName]
    }
    def env = getEnvByName(baseUrl, params['Environment'].toString())
    if (env == null) {
        throw new NoSuchElementException("The '${params['Environment']}' environment does not exist")
    }
    return env
}

def getDefaultEnv(String baseUrl) {
    URL url = new URL("${baseUrl}/projects/${params['Project ID']}/environments/default")
    def environment = getRequest(url, "Default Environment")
    return [environment["Id"], environment["Name"]]
}

def getEnvByName(String baseUrl, String envName) {
    String cursor = ""
    boolean fetchEnvironmentPage = true
    while (fetchEnvironmentPage) {
        URL url = addQueryParameterToUrl("${baseUrl}/projects/${params['Project ID']}/environments", [
            q: envName,
            cursor: cursor
        ])
        def environments = getRequest(url, "Environment")
        for (environment in environments["Items"]) {
            if (environment["Name"].toString() == envName) {
                return [environment["Id"], environment["Name"]]
            }
        }
        cursor = environments.Info.Next
        fetchEnvironmentPage = environments.Info.HasNext
    }
    return null
}

def getRequest(URL url, String requestedFor) {
    HttpURLConnection conn = url.openConnection()
    conn.setRequestMethod("GET")
    conn.setDoInput(true)
    conn.setRequestProperty("Authorization", "APIKEY $useMangoApiKey")
    conn.connect()
    if (conn.responseCode == 200) {
        String content = conn.getInputStream().getText()
        return new JsonSlurper().parseText(content)
    }
    throw new Exception("${requestedFor} GET request failed with code: ${conn.responseCode}")
}

pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Run useMango Tests') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'useMangoApiKey', variable: 'useMangoApiKey')]) {
                        String TEST_SERVICE_URL = "https://tests.api.usemango.co.uk/v1"
                        String SCRIPTS_SERVICE_URL = "https://scripts.api.usemango.co.uk/v1"
                        String APP_WEBSITE_URL = "https://app.usemango.co.uk"
                        
                        // Define log file path
                        def logFile = 'logs.txt'
                        def passed = 0
                        def failed = 0
                        def allPassed = true
                        def testResults = [:]
                        Integer count = 0
                        
                        echo "Running tests in project ${params['Project ID']} with tags ${params['Tags']}"
                        def (envId, envName) = getEnvIdAndName(TEST_SERVICE_URL)
                        def tests = getTests(TEST_SERVICE_URL)
                        def testJobs = [:]

                        tests.eachWithIndex { test, index -> 
                            echo "Scheduling ${test.Name}"
                            testJobs[test.Id] = {
                                node('usemango') {
                                    wrap([$class: "MaskPasswordsBuildWrapper", varPasswordPairs: [[password: '%useMangoApiKey%']]]) {
                                        dir("${env.WORKSPACE}\\${tests[index].Id}") {
                                            deleteDir()
                                        }
                                        dir("${env.WORKSPACE}\\${tests[index].Id}") {
                                            List<Map> scenarioList = tests[index].Scenarios
                                            String datasetType = ""
                                            Map paramMap = [:]
                                            boolean isMultiDataset = false
                                            
                                            if (scenarioList != null) {
                                                if (scenarioList.size() == 1) {
                                                    datasetType = "Default Dataset"
                                                } else {
                                                    isMultiDataset = true
                                                    datasetType = "Multi Dataset 'Dataset Count=${scenarioList.size()}'"
                                                }
                                                paramMap["scenario"] = scenarioList.collect { it.Id }
                                            }
                                            paramMap["environment"] = envId
                                            String url = addQueryParameterToUrl(SCRIPTS_SERVICE_URL + "/tests/" + tests[index].Id.toString(), paramMap).toString()
                                            
                                            bat "curl -s --create-dirs -L -D \"response.txt\" -X GET \"${url}\" -H \"Authorization: APIKEY " + '%useMangoApiKey%' +"\" --output \"${tests[index].Id}.pyz\""
                                            String httpCode = powershell(returnStdout: true, script: "Write-Output (Get-Content \"response.txt\" | select -First 1 | Select-String -Pattern '.*HTTP/1.1 ([^\\\"]*) *').Matches.Groups[1].Value")
                                            echo "Test executable response code - ${httpCode}"

                                            try {
                                                // Check if the HTTP response code is 200 OK
                                                if (httpCode.contains("200")) {
                                                    echo "Executing - '${tests[index].Name}' ${datasetType}"
                                            
                                                    // Execute the test command using the test ID and useMango API key
                                                    bat "\"%UM_PYTHON_PATH%\" ${tests[index].Id}.pyz -k '%useMangoApiKey%' -j result.xml"
                                            
                                                    // Read the result from the result.xml file
                                                    def result = readXML file: 'result.xml'
                                            
                                                    // Get the run ID (for tracking test execution)
                                                    def run_id = getRunId()
                                            
                                                    // Check if the run_id is valid (non-null)
                                                    if (run_id != null) {
                                                        // If run_id exists, mark test as passed
                                                        testResults.put(count, "TestName: '${tests[index].Name}' ${datasetType} (Passed)")
                                                        passed++ // Increment passed tests counter
                                                    } else {
                                                        // If no run_id found, mark test as failed with specific message
                                                        testResults.put(count, "TestName: '${tests[index].Name}' ${datasetType} (Failed) - No RunId found")
                                                        failed++ // Increment failed tests counter
                                                    }
                                                } else {
                                                    // HTTP error occurred, mark test as failed with HTTP error message
                                                    testResults.put(count, "TestName: '${tests[index].Name}' ${datasetType} (Failed) - HTTP Error ${httpCode}")
                                                    failed++ // Increment failed tests counter
                                                }
                                            } catch (Exception e) {
                                                // Catch any other exceptions, log them and mark the test as failed
                                                testResults.put(count, "TestName: '${tests[index].Name}' ${datasetType} (Failed) - Error: ${e.getMessage()}")
                                                failed++ // Increment failed tests counter
                                                echo "Error during execution of test '${tests[index].Name}': ${e.getMessage()}"
                                            }

                                        }
                                    }
                                }
                            }
                            count++
                        }

                        parallel testJobs

                        // Final Results Logging
                        def resultMessage = "Test Results:\n"
                        testResults.each { result ->
                            resultMessage += result + "\n"
                        }
                        
                        writeFile file: logFile, text: resultMessage
                        archiveArtifacts artifacts: logFile, allowEmptyArchive: true
                        echo "Test run completed. Passed: ${passed}, Failed: ${failed}"
                    }
                }
            }
        }
    }
}
