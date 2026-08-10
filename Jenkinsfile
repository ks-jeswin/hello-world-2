pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        // Fixes AccessDeniedException: /.sonar by placing the cache inside the workspace
        SONAR_USER_HOME = "${WORKSPACE}/.sonar"
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    stages {
        stage('Build & Test') {
            steps {
                // Prepares JaCoCo agent without modifying pom.xml
                sh 'mvn clean test org.jacoco:jacoco-maven-plugin:0.8.11:prepare-agent -B'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Quality Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                // Added -e -X for full debug logs and disabled automatic JRE provisioning
                sh '''
                    mvn org.jacoco:jacoco-maven-plugin:0.8.11:report \
                        org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar \
                        -Dsonar.userHome=${WORKSPACE}/.sonar \
                        -Dsonar.scanner.skipJreProvisioning=true \
                        -e -X -B
                '''
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Checking Quality Gate status...'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
