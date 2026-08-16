pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args  '-v $HOME/.m2:/root/.m2 --network host'
        }
    }
    environment {
        APP_NAME     = 'hello-world-2'
        APP_VERSION  = "1.0.${env.BUILD_NUMBER}"
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        ansiColor('xterm')
    }
    triggers {
        githubPush()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH} | Commit: ${env.GIT_COMMIT[0..7]}"
            }
        }
        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
            post {
                success { echo 'Compile successful - moving to Test stage.' }
                failure { echo 'Compile FAILED - check pom.xml and source errors.' }
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test -B'
            }
            post {
                always {
                    junit(testResults: 'target/surefire-reports/**/*.xml',
                          allowEmptyResults: false)
                }
                unstable {
                    echo 'WARNING: Tests failed - build marked UNSTABLE.'
                    script {
                        def results = currentBuild.rawBuild.getAction(
                            hudson.tasks.test.AbstractTestResultAction.class)
                        if (results) {
                            def passRate = (results.totalCount - results.failCount) /
                                           results.totalCount * 100
                            if (passRate < 80) {
                                error("Test pass rate ${passRate.round(1)}% is below 80% threshold!")
                            }
                        }
                    }
                }
            }
        }
        stage('Quality Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Local') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.10.0.2594:sonar \
                              -Dsonar.projectKey=${env.APP_NAME} \
                              -Dsonar.projectName="TechBuild ${env.APP_NAME}" \
                              -Dsonar.projectVersion=${env.APP_VERSION} \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.token=\$SONAR_TOKEN \
                              -B
                        """
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Package & Archive') {
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts(artifacts: 'target/*.war', fingerprint: true)
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.war"
            }
        }
        stage('Publish Artifact') {
            when { branch 'master' }
            steps {
                nexusArtifactUploader(
                    nexusVersion:  'nexus3',
                    protocol:      'http',
                    nexusUrl:      'localhost:8081',
                    groupId:       'io.techbuild',
                    version:       env.APP_VERSION,
                    repository:    'techbuild-releases',
                    credentialsId: 'nexus-creds',
                    artifacts: [[
                        artifactId: env.APP_NAME,
                        classifier: '',
                        file:       "target/${env.APP_NAME}-${env.APP_VERSION}.war",
                        type:       'war'
                    ]]
                )
            }
        }
    }
    post {
        success {
            withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' \
                    --data '{"text":"SUCCESS: ${env.APP_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
        failure {
            withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' \
                    --data '{"text":"FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
        unstable {
            withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' \
                    --data '{"text":"UNSTABLE: ${env.APP_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
    }
}
