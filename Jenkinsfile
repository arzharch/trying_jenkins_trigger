#!groovy
import groovy.json.JsonSlurper

node {
    stage('Log Outputs') {
        // Define the path for logs
        def logFilePath = 'C:/Users/arsh.chowhan/Desktop/logs.txt'

        // Ensure the log file exists
        writeFile(file: logFilePath, text: '')

        // Log test execution summary
        echo "Logging output to ${logFilePath}"

        // Example log entry
        def testResults = [
            [Name: 'Test1', Status: 'Passed', DatasetType: 'Default Dataset'],
            [Name: 'Test2', Status: 'Failed', DatasetType: 'Multi Dataset']
        ]

        // Write results to the log file
        testResults.each { result ->
            def logMessage = "Test: ${result.Name}, Status: ${result.Status}, Dataset Type: ${result.DatasetType}"
            echo logMessage
            appendToLog(logFilePath, logMessage)
        }

        // Summarize results
        def passed = testResults.count { it.Status == 'Passed' }
        def failed = testResults.count { it.Status == 'Failed' }

        def summaryMessage = "Total Tests: ${testResults.size()}, Passed: ${passed}, Failed: ${failed}"
        echo summaryMessage
        appendToLog(logFilePath, summaryMessage)
    }
}

// Function to append logs to the specified log file
def appendToLog(filePath, message) {
    writeFile(file: filePath, text: "${message}\n", append: true)
}
