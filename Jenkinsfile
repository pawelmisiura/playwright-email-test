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
             sh 'npx playwright test'
          }
       }
    }
}