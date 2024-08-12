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
                }
            }
        }
    }
    
    post {
        always {
            // Archive test results
            writeFile file: 'test-output.txt', text: env.TEST_OUTPUT
            archiveArtifacts artifacts: 'test-output.txt', fingerprint: true
        }
        failure {
            script {
                // Parse output to find failed tests and their team tags
                def failedTests = parseFailedTests(env.TEST_OUTPUT)
                
                // Group failed tests by team
                def teamFailures = groupTestsByTeam(failedTests)
                
                // Send emails to teams with failures
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
        to: "pawelmisiura1@gmail.com",
        mimeType: 'text/plain'
    )
}