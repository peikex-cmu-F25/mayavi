pipeline {
    agent any
    stages {
        stage('1. Checkout') {
            steps {
                git credentialsId: 'github-token', 
                    url: 'https://github.com/peikex-cmu-F25/mayavi.git', 
                    branch: 'main'
            }
        }
        stage('2. SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    
                    withSonarQubeEnv('SonarQube') { 
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=mayavi-project \
                            -Dsonar.sources=. \
                            -Dsonar.language=py"
                    }
                }
            }
        }
        stage('3. Check Quality Gate') {
            steps {
                    waitForQualityGate abortPipeline: true   
            }
        }
    }
}