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
                sh 'poetry install --no-root'
            }
        }

        stage('Unit and Integration Tests') {
            steps {
                echo 'Running tests with pytest...'
                sh 'poetry run pytest || true'
            }
        }

        stage('Code Analysis') {
            steps {
                echo 'Running code analysis...'
                sh '''
                    pip install flake8 --break-system-packages || true
                    flake8 . --max-line-length=120 --exit-zero
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    pip install bandit safety --break-system-packages || true
                    mkdir -p reports
                    bandit -r . -f json -o reports/bandit-report.json || true
                    safety check --output json > reports/safety-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
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
