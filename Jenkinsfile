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
                    // Run Playwright tests and write output to file
                    def exitCode = sh(script: 'npx playwright test --reporter=list > test-output.txt 2>&1', returnStatus: true)
                    env.TEST_OUTPUT = readFile('test-output.txt')
                    env.TEST_EXIT_CODE = exitCode.toString()

                    echo "Test Exit Code: ${env.TEST_EXIT_CODE}"
                    echo "First 1000 characters of Test Output: ${env.TEST_OUTPUT.take(1000)}"

                    if (exitCode != 0) {
                        error "Tests failed with exit code ${exitCode}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'test-output.txt', fingerprint: true
        }
        failure {
            script {
                def failedTests = parseFailedTests(env.TEST_OUTPUT)
                def teamFailures = groupTestsByTeam(failedTests)
                teamFailures.each { team, tests ->
                    sendEmailToTeam(team, tests)
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
    )
}