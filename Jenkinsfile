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
                    env.TEST_OUTPUT = sh(script: 'npx playwright test --reporter=list', returnStdout: true)
                    
                    // Add a check to see if we captured any output
                    if (env.TEST_OUTPUT?.trim()) {
                        echo "Test output captured successfully"
                    } else {
                        echo "Warning: No test output captured"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                if (env.TEST_OUTPUT?.trim()) {
                    writeFile file: 'test-output.txt', text: env.TEST_OUTPUT
                    archiveArtifacts artifacts: 'test-output.txt', fingerprint: true
                } else {
                    echo "No test output to archive"
                }
            }
        }
        failure {
            script {
                if (env.TEST_OUTPUT?.trim()) {
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
    if (!output?.trim()) {
        echo "No output to parse for failed tests"
        return []
    }
    def failedTests = []
    def currentTag = ""
    def lines = output.split('\n')
    lines.each { line ->
        def tagMatcher = line =~ /describe\(".*", \{ tag: "(.*)" \}/
        if (tagMatcher.find()) {
            currentTag = tagMatcher.group(1)
        }
        def failedTestMatcher = line =~ /✘\s+(.+)\s+\(\d+ms\)/
        if (failedTestMatcher.find()) {
            failedTests << [name: failedTestMatcher.group(1), team: currentTag]
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