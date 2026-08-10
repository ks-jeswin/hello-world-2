pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17-alpine'
            args '-v $HOME/.m2:/root/.m2'
        }
    }

    environment {
        APP_NAME = 'hello-world-2'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    echo "Branch: ${env.GIT_BRANCH} | Commit: ${env.GIT_COMMIT?.take(8)}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v1.0.${BUILD_NUMBER}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
        }

        stage('Test') {
            steps {
                echo "Running unit tests and generating JaCoCo coverage report..."
                sh 'mvn test jacoco:report -B'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Quality Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube-Local') {
                        sh """
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=${env.APP_NAME} \
                            -Dsonar.projectName='TechBuild ${env.APP_NAME}' \
                            -Dsonar.projectVersion=1.0.${BUILD_NUMBER} \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                            -B
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to Quality Gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
