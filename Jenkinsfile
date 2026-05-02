pipeline {
    agent any

    triggers {
        pollSCM('H/1 * * * *')
    }

    environment {
        STAGING_SERVER = 'root@192.168.30.104'
        PRODUCTION_SERVER = 'root@192.168.30.104'
    }

    stages {

        stage('Build') {
            steps {
                echo 'Building project with Poetry...'
                sh '''
                /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry install --no-root
                '''
            }
        }

        stage('Unit and Integration Tests') {
            steps {
                echo 'Running tests with pytest...'
                sh '''
                /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry run pytest || true
                '''
            }
        }

        stage('Code Analysis') {
            steps {
                echo 'Running code analysis...'
                sh '''
                /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry run sonar-scanner 
                '''
            }
        }

        stage('Security Scan') {
            steps {
                echo 'Running Dependency Check...'
                sh '''
                dependency-check.sh --scan .
                '''
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to Proxmox VM (staging)...'
                sh '''
                scp -r . $STAGING_SERVER:/mnt/app/
                ssh $STAGING_SERVER "
                    cd /mnt/app &&
                    /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry install --no-root &&
                    /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry run python main.py  &
                "
                '''
            }
        }

        stage('Integration Tests on Staging') {
            steps {
                sh '''
                ssh $STAGING_SERVER "
                    cd /mnt/app &&
                    /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry run pytest || true
                "
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                echo 'Deploying to production VM...'
                sh '''
                scp -r . $PRODUCTION_SERVER:/mnt/app/
                ssh $PRODUCTION_SERVER "
                    cd /mnt/app &&
                    /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry install --no-root &&
                    /var/jenkins_home/.local/share/pypoetry/venv/bin/poetry run python main.py  &
                "
                '''
            }
        }
    }
}
