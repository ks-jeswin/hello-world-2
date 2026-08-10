pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17-alpine'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Branch: ${env.GIT_BRANCH} | Commit: ${env.GIT_COMMIT?.take(8)}"
                }
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building hello-world-2...'
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running unit tests and generating JaCoCo coverage report...'
                // Using fully qualified plugin name prevents 'No plugin found for prefix jacoco' error
                sh 'mvn test org.jacoco:jacoco-maven-plugin:0.8.11:report -B'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Quality Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -B'
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...'
                // Add your Quality Gate check step here (e.g., waitForQualityGate)
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
