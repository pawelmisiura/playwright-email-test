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
        stage('Extract Tags') {
            steps {
                script {
                    env.TEST_TAGS = sh(script: '''
                        for file in tests/*.spec.js; do
                            tag=$(grep 'test.describe.*tag:' "$file" | sed -E 's/.*tag: "([^"]+)".*/\\1/')
                            echo "$file:$tag"
                        done
                    ''', returnStdout: true).trim()
                    echo "Extracted tags: ${env.TEST_TAGS}"
                }
            }
        }
        stage('Run Tests') {
            steps {
                script {
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
                def failedTests = parseFailedTests(env.TEST_OUTPUT, env.TEST_TAGS)
                echo "Failed Tests: ${failedTests}"
                def teamFailures = groupTestsByTeam(failedTests)
                echo "Team Failures: ${teamFailures}"
                teamFailures.each { team, tests ->
                    sendEmailToTeam(team, tests)
                }
            }
        }
    }
}

def parseFailedTests(output, tags) {
    def failedTests = []
    def tagMap = [:]
    tags.split('\n').each { 
        def parts = it.split(':')
        if (parts.size() == 2) {
            tagMap[parts[0].split('/')[-1]] = parts[1]
        }
    }
    def lines = output.split('\n')
    lines.each { line ->
        def failedTestMatcher = line =~ /✘\s+\d+\s+\[chromium\]\s+›\s+(.*?):.*?›\s+(.*?)\s+›\s+(.*?)\s+\(\d+.*\)/
        if (failedTestMatcher.find()) {
            def fileName = failedTestMatcher.group(1)
            def testSuite = failedTestMatcher.group(2)
            def testName = failedTestMatcher.group(3)
            def team = tagMap[fileName] ?: 'unknown'
            failedTests << [name: "${fileName} › ${testSuite} › ${testName}", team: team]
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
    def subject = "Test Failures for ${team}"
    def body = "The following tests failed for ${team}:\n${tests.join('\n')}"
    def to = "${team.replaceAll('@', '')}@example.com"
    
    echo "Preparing to send email:"
    echo "To: ${to}"
    echo "Subject: ${subject}"
    echo "Body:\n${body}"
    
    emailext (
        subject: subject,
        body: body,
        to: to,
        mimeType: 'text/plain'
    )
}