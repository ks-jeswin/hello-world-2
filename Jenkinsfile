// Repository: https://github.com/ks-jeswin/hello-world-2.git
// Pipeline: Checkout → Build → Test → Quality Analysis → Quality Gate → Package & Archive → Publish Artifact

pipeline {

    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17-alpine'
            args  '-v $HOME/.m2:/root/.m2'
        }
    }

    environment {
        APP_NAME     = 'hello-world-2'
        APP_VERSION  = "1.0.${env.BUILD_NUMBER}"
        MAVEN_OPTS   = '-Xmx1024m -XX:+TieredCompilation'
        SONAR_URL    = 'http://sonarqube:9000'
        ARTIFACT_DIR = 'target'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH ?: 'main'} | Commit: ${env.GIT_COMMIT ? env.GIT_COMMIT[0..7] : 'HEAD'}"
            }
        }

        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
            post {
                success { echo 'Compile successful — moving to Test stage.' }
                failure { echo 'Compile FAILED — check pom.xml and source errors.' }
            }
        }

        stage('Test') {
            steps {
                // Generates JaCoCo report alongside test execution
                sh 'mvn test jacoco:report -B'
            }
            post {
                always {
                    junit(testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: false)
                }
                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'
                    script {
                        def results = currentBuild.rawBuild.getAction(hudson.tasks.test.AbstractTestResultAction.class)
                        if (results) {
                            def passRate = (results.totalCount - results.failCount) / results.totalCount * 100
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
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                    withSonarQubeEnv('SonarQube-Local') {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=${env.APP_NAME} \
                              -Dsonar.projectName="TechBuild ${env.APP_NAME}" \
                              -Dsonar.projectVersion=${env.APP_VERSION} \
                              -Dsonar.token=${SONAR_AUTH_TOKEN} \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -B
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            // Removed erroneous "agent none" directive
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package & Archive') {
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts(artifacts: 'target/*.jar', fingerprint: true)
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.jar"
            }
        }

        stage('Publish Artifact') {
            when { branch 'main' }
            steps {
                nexusArtifactUploader(
                    nexusVersion:  'nexus3',
                    protocol:      'http',
                    nexusUrl:      'host.docker.internal:8081', // Corrected loopback endpoint
                    groupId:       'io.techbuild',
                    version:       env.APP_VERSION,
                    repository:    'techbuild-releases',
                    credentialsId: 'nexus-creds',
                    artifacts: [[
                        artifactId: env.APP_NAME,
                        classifier: '',
                        file:       "target/${env.APP_NAME}-${env.APP_VERSION}.jar",
                        type:       'jar'
                    ]]
                )
            }
        }

    }

    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
        }
        always {
            cleanWs()
        }
    }
}
