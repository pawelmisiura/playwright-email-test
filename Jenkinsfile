pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS' // Use the name you've configured in Jenkins for Node.js
    }
    
    stages {
        stage('e2e-tests') {
            steps {
                sh 'npm ci'
                sh 'npx playwright install'
                script {
                    try {
                        // Run Playwright tests and capture output
                        def testOutput = sh(script: 'npx playwright test --reporter=list', returnStdout: true)
                        
                        // Parse output to find failed tests and their team tags
                        def failedTests = parseFailedTests(testOutput)
                        
                        // Group failed tests by team
                        def teamFailures = groupTestsByTeam(failedTests)
                        
                        // Archive test results
                        writeFile file: 'test-output.txt', text: testOutput
                        archiveArtifacts artifacts: 'test-output.txt', fingerprint: true

                        // Send emails to teams with failures
                        if (!teamFailures.isEmpty()) {
                            teamFailures.each { team, tests ->
                                sendEmailToTeam(team, tests)
                            }
                        } else {
                            echo "All tests passed. No emails sent."
                        }
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        error("Test execution failed: ${e.message}")
                    }
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