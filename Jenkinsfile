pipeline {
    agent any

    stages {
        stage('Development') {
            steps {
                echo 'I am in development stage'
                sh 'git --version'
    
            }
        }
        stage('testing') {
            steps {
                echo 'I am in testing stage'
                sh 'docker --version'
            }
        }
        stage('production') {
            steps {
                echo 'I am in production stage'
                sh 'java --version'
            }
        }
    }
}
