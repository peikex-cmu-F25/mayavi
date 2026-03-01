pipeline {
    agent any
    environment {
        DATAPROC_CLUSTER = 'hadoop-cluster'
        DATAPROC_REGION = 'us-central1'
        GCS_BUCKET = 'code-quality-assurance-project-hadoop-bucket'
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
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') { 
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=mayavi-project -Dsonar.sources=. -Dsonar.language=py"
                    }
                }
            }
        }
        stage('3. Check Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        stage('4. Deploy to Hadoop') {
            steps {
                echo "Quality Gate Passed with 0 Blockers. Deploying to Hadoop..."
                sh 'echo "Running Terraform apply..." '
                sh '''
                    # init cluster
                    gcloud dataproc clusters start ${DATAPROC_CLUSTER} --region=${DATAPROC_REGION}
                    
                    # upload repo file to GCS
                    gsutil -m cp -r . gs://${GCS_BUCKET}/input/
                    
                    # upload MapReduce job script
                    gsutil cp mapreduce/line_count.py gs://${GCS_BUCKET}/
                    
                    # submit Hadoop streaming job
                    gcloud dataproc jobs submit hadoop \
                    --cluster=${DATAPROC_CLUSTER} \
                    --region=${DATAPROC_REGION} \
                    --jar=/usr/lib/hadoop-mapreduce/hadoop-mapreduce-streaming.jar \
                    -- \
                    -input gs://${GCS_BUCKET}/input/ \
                    -output gs://${GCS_BUCKET}/output/ \
                    -mapper "python3 line_count.py mapper" \
                    -reducer "python3 line_count.py reducer" \
                    -file gs://${GCS_BUCKET}/line_count.py
                '''
            }
        }
    }
}