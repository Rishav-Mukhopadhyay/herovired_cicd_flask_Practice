pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
        NOTIFICATION_EMAIL = 'rishav1994sonai@gmail.com'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
                sh '''
                    python3 -m venv ${VENV_DIR}
                    ${VENV_DIR}/bin/pip install --upgrade pip
                    ${VENV_DIR}/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Starting a throwaway MongoDB container for the test suite...'
                sh '''
                    docker rm -f ci-mongo >/dev/null 2>&1 || true
                    docker run -d --name ci-mongo -p 27017:27017 mongo:7
                    sleep 5
                '''
                withEnv([
                    "MONGO_URI=mongodb://localhost:27017/test_student_db",
                    "SECRET_KEY=jenkins-test-key"
                ]) {
                    sh '''
                        ${VENV_DIR}/bin/pytest test_app.py --junitxml=junit/test-results.xml
                    '''
                }
            }
            post {
                always {
                    sh 'docker rm -f ci-mongo >/dev/null 2>&1 || true'
                    junit 'junit/test-results.xml'
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to the staging environment...'
                withCredentials([
                    string(credentialsId: 'mongo-uri', variable: 'MONGO_URI'),
                    string(credentialsId: 'flask-secret-key', variable: 'SECRET_KEY')
                ]) {
                    sh '''
                        pkill -f "${WORKSPACE}/${VENV_DIR}/bin/python app.py" || true
                        echo "MONGO_URI=${MONGO_URI}" > .env
                        echo "SECRET_KEY=${SECRET_KEY}" >> .env
                        nohup ${VENV_DIR}/bin/python app.py > flask_app.log 2>&1 &
                        sleep 5
                        curl -f http://localhost:5000
                    '''
                }
            }
        }
    }

    post {
        success {
            emailext to: "${env.NOTIFICATION_EMAIL}",
                     subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: "The pipeline completed successfully.\n\nBuild details: ${env.BUILD_URL}"
        }
        failure {
            emailext to: "${env.NOTIFICATION_EMAIL}",
                     subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: "The pipeline failed. See the attached application log and console output.\n\n${env.BUILD_URL}",
                     attachmentsPattern: "flask_app.log"
        }
    }
}
