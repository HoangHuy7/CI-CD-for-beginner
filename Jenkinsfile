pipeline{
    agent any
    tools{
        maven 'Maven 3.9.6'
    }
    stages{
        stage('Checkout'){
            steps{
                echo 'Checkout the code'
                checkout scm
            }
        }
        stage('Build'){
            steps{
                echo 'Build the code'
                dir('backend') {
                    sh 'mvn clean package'
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Pipeline successfully completed.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
