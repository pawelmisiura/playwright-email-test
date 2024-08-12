pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS' // Use the name you've configured in Jenkins for Node.js
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
                sh 'npx playwright install'
            }
        }
        stage('Run Tests') {
            steps {
                script {
                    // Run Playwright tests and capture output
                    def testResult = sh(script: 'npx playwright test --reporter=list', returnStatus: true, returnStdout: true)
                    env.TEST_OUTPUT = testResult.toString()
                    env.TEST_EXIT_CODE = testResult

                    // Print the captured output for debugging
                    echo "Test Output: ${env.TEST_OUTPUT}"
                    echo "Test Exit Code: ${env.TEST_EXIT_CODE}"

                    // Fail the build if tests failed
                    if (env.TEST_EXIT_CODE != '0') {
                        error "Tests failed with exit code ${env.TEST_EXIT_CODE}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                if (env.TEST_OUTPUT) {
                    writeFile file: 'test-output.txt', text: env.TEST_OUTPUT
                    archiveArtifacts artifacts: 'test-output.txt', fingerprint: true
                } else {
                    echo "No test output to archive"
                }
            }
        }
        failure {
            script {
                if (env.TEST_OUTPUT) {
                    def failedTests = parseFailedTests(env.TEST_OUTPUT)
                    def teamFailures = groupTestsByTeam(failedTests)
                    teamFailures.each { team, tests ->
                        sendEmailToTeam(team, tests)
                    }
                } else {
                    echo "No test output available for failure analysis"
                }
            }
        }
    }
}

def parseFailedTests(output) {
    def failedTests = []
    def currentTag = ""
    def lines = output.split('\n')
    lines.each { line ->
        def tagMatcher = line =~ /describe\(".*", \{ tag: "(.*)" \}/
        if (tagMatcher.find()) {
            currentTag = tagMatcher.group(1)
        }
        def failedTestMatcher = line =~ /✘\s+(\d+)\s+\[.*\]\s+›\s+(.*)\s+\(\d+.*\)/
        if (failedTestMatcher.find()) {
            failedTests << [name: failedTestMatcher.group(2), team: currentTag]
        }
    }
    return failedTests
}

def groupTestsByTeam(failedTests) {
    def teamFailures = [:]
    failedTests.each { test ->
        if (!teamFailures.containsKey(test.team)) {
            teamFailures[test.team] = []
        }
        teamFailures[test.team] << test.name
    }
    return teamFailures
}

def sendEmailToTeam(team, tests) {
    emailext (
        subject: "Test Failures for ${team}",
        body: "The following tests failed for ${team}:\n${tests.join('\n')}",
        to: "${team}@example.com",
        mimeType: 'text/plain'