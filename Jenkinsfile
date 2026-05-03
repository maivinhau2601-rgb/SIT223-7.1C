pipeline {
    agent any
    triggers {
        pollSCM('H/1 * * * *')
    }
    environment {
        STAGING_SERVER    = 'root@192.168.30.104'
        PRODUCTION_SERVER = 'root@192.168.30.104'
    }
    stages {

        stage('Build') {
            steps {
                echo 'Building project with Poetry...'
                sh '''
                poetry lock 
                poetry install --no-root
                '''
            }
        }

        stage('Unit and Integration Tests') {
            steps {
                echo 'Running tests with pytest...'
                sh 'poetry run pytest --junitxml=reports/test-report.xml 2>&1 | tee reports/test-logs.txt || true'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/test-logs.txt', allowEmptyArchive: true
        
                    emailext(
                        to: 'maivinhau2601@gmail.com',
                        subject: "Test Stage: ${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """
                            <h2>Unit and Integration Test Result: ${currentBuild.currentResult}</h2>
                            <p><b>Job:</b> ${env.JOB_NAME}</p>
                            <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                            <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        """,
                        attachmentsPattern: 'reports/test-logs.txt, reports/test-report.xml',
                        mimeType: 'text/html'
                    )
                }
            }
        }

        stage('Code Analysis') {
            steps { 
                echo 'Running code analysis...'
                sh '''
                    poetry add --group dev flake8 --quiet
                    poetry run flake8 . --max-line-length=120 --exit-zero
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    mkdir -p reports
                    poetry run bandit -r . -f json -o reports/bandit-report.json || true
                    poetry run safety check --output json > reports/safety-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        
                    emailext(
                        to: 'maivinhau2601@gmail.com,
                        subject: "Security Scan: ${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """
                            <h2>Security Scan Result: ${currentBuild.currentResult}</h2>
                            <p><b>Job:</b> ${env.JOB_NAME}</p>
                            <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                            <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        """,
                        attachmentsPattern: 'reports/bandit-report.json, reports/safety-report.json',
                        mimeType: 'text/html'
                    )
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to staging...'
                sh '''
                    scp -o StrictHostKeyChecking=no -r . $STAGING_SERVER:/mnt/app/
                    ssh -o StrictHostKeyChecking=no $STAGING_SERVER "
                        export PATH=/root/.local/bin:$PATH &&
                        cd /mnt/app &&
                        poetry install --no-root &&
                        pkill -f 'python3 main.py' || true &&
                        nohup python3 main.py > /var/log/app.log 2>&1 &
                    "
                '''
            }
        }

        stage('Integration Tests on Staging') {
            steps {
                sh '''
                    ssh -o StrictHostKeyChecking=no $STAGING_SERVER "
                        export PATH=/root/.local/bin:$PATH &&
                        cd /mnt/app &&
                        poetry run pytest || true
                    "
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                echo 'Deploying to production...'
                sh '''
                    scp -o StrictHostKeyChecking=no -r . $PRODUCTION_SERVER:/mnt/app/
                    ssh -o StrictHostKeyChecking=no $PRODUCTION_SERVER "
                        export PATH=/root/.local/bin:$PATH &&
                        cd /mnt/app &&
                        poetry install --no-root &&
                        pkill -f 'python3 main.py' || true &&
                        nohup python3 main.py > /var/log/app.log 2>&1 &
                    "
                '''
            }
        }
    }
}
