pipeline {
    agent any

    environment {
        GCP_PROJECT      = "code-quality-assurance-project"
        DATAPROC_CLUSTER = "hadoop-cluster"
        DATAPROC_REGION  = "us-central1"
        GCS_BUCKET       = "code-quality-assurance-project-hadoop-bucket"
    }

    options {
        timeout(time: 150, unit: 'MINUTES')
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
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=mayavi-project \
                              -Dsonar.sources=. \
                              -Dsonar.language=py
                        """
                    }
                }
            }
        }

        stage('3. Quality Gate & Hadoop') {
            steps {
                script {
                    def qg
                    timeout(time: 5, unit: 'MINUTES') {
                        qg = waitForQualityGate()
                    }
                    if (qg.status == 'OK') {
                        echo "No blocker issues. Submitting Hadoop job..."
                        timeout(time: 120, unit: 'MINUTES') {
                            sh """
                                OUTPUT_PATH=gs://${GCS_BUCKET}/output/\$(date +%s)/
                                gcloud dataproc jobs submit hadoop \
                                  --cluster=${DATAPROC_CLUSTER} \
                                  --region=${DATAPROC_REGION} \
                                  --jar=file:///usr/lib/hadoop/hadoop-streaming.jar \
                                  --files=gs://${GCS_BUCKET}/line_count.py \
                                  --properties=mapreduce.input.fileinputformat.input.dir.recursive=true \
                                  -- \
                                  -input gs://${GCS_BUCKET}/input/ \
                                  -output \${OUTPUT_PATH} \
                                  -mapper 'python3 line_count.py mapper' \
                                  -reducer 'python3 line_count.py reducer'
                                echo "Job Results:"
                                gsutil cat \${OUTPUT_PATH}part-00000
                            """
                        }
                    } else {
                        echo "Blocker issues found. Hadoop job will NOT run."
                        echo "SonarQube status: ${qg.status}"
                    }
                }
            }
        }

    } 

    post {
        always {
            echo "Pipeline complete."
        }
    }
}