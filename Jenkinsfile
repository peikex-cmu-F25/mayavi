pipeline {
    agent any
    environment {
        DATAPROC_CLUSTER = 'hadoop-cluster'
        DATAPROC_REGION  = 'us-central1'
        GCS_BUCKET       = 'code-quality-assurance-project-hadoop-bucket'
    }
    options {
        timeout(time: 20, unit: 'MINUTES')
    }
    stages {
        stage('1. Checkout') {
            steps {
                git url: 'https://github.com/peikex-cmu-F25/mayavi.git', branch: 'main'
            }
        }
        stage('2. SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarqube-scanner'
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=mayavi-project \
                            -Dsonar.sources=. \
                            -Dsonar.language=py"
                    }
                }
            }
        }
        stage('3. Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        stage('4. Run Hadoop Job') {
            steps {
                sh """
                    gcloud dataproc jobs submit hadoop \
                        --cluster=${DATAPROC_CLUSTER} \
                        --region=${DATAPROC_REGION} \
                        --jar=file:///usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar \
                        -- wordcount \
                        gs://${GCS_BUCKET}/input \
                        gs://${GCS_BUCKET}/output/\$(date +%s)
                """
            }
        }
    }
    post {
        always {
            echo 'Pipeline finished'
        }
        failure {
            echo 'Pipeline failed - either quality gate blocked or Hadoop job failed'
        }
    }
}