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
        MIN_PASS_RATE = 80
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        ansiColor('xterm') // requires AnsiColor plugin
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH ?: 'main'} | Commit: ${env.GIT_COMMIT ? env.GIT_COMMIT[0..7] : 'HEAD'}"
                sh 'git log --oneline -5'
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
                // Generates JaCoCo report alongside test execution.
                // catchError lets junit still publish results even if tests fail,
                // while marking the stage UNSTABLE so we can evaluate pass rate below.
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    // Fully-qualified plugin coordinates used instead of the
                    // 'jacoco' prefix shorthand, since prefix resolution
                    // fails with "No plugin found for prefix 'jacoco'" unless
                    // org.jacoco:jacoco-maven-plugin is declared in pom.xml.
                    // Best fix long-term is to add the plugin to pom.xml,
                    // but this works regardless of pom.xml configuration.
                    sh 'mvn test org.jacoco:jacoco-maven-plugin:0.8.12:report -B'
                }
            }
            post {
                always {
                    junit(testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: false)
                }
                unstable {
                    script {
                        def results = currentBuild.rawBuild.getAction(hudson.tasks.test.AbstractTestResultAction.class)
                        if (results && results.totalCount > 0) {
                            def passRate = (results.totalCount - results.failCount) / results.totalCount * 100
                            echo "Test pass rate: ${passRate.round(1)}% (threshold: ${env.MIN_PASS_RATE}%)"
                            if (passRate < env.MIN_PASS_RATE.toInteger()) {
                                error("Test pass rate ${passRate.round(1)}% is below ${env.MIN_PASS_RATE}% threshold!")
                            } else {
                                echo 'WARNING: Some tests failed, but pass rate is within acceptable threshold — continuing.'
                            }
                        } else {
                            echo 'WARNING: Test stage unstable and no test result data found — treating as failure.'
                            error('No test results available to evaluate pass rate.')
                        }
                    }
                }
            }
        }

        stage('Quality Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                    withSonarQubeEnv('SonarQube-Local') {
                        // Token passed via -Dsonar.login read from env inside sh,
                        // not interpolated into the script text, to avoid leaking
                        // it into build logs if verbose/debug output is enabled.
                        sh '''
                            mvn sonar:sonar \
                              -Dsonar.projectKey=${APP_NAME} \
                              -Dsonar.projectName="TechBuild ${APP_NAME}" \
                              -Dsonar.projectVersion=${APP_VERSION} \
                              -Dsonar.token=${SONAR_AUTH_TOKEN} \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -B
                        '''
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
                archiveArtifacts(artifacts: 'target/*.jar', fingerprint: true)
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.jar"
            }
        }

        stage('Publish Artifact') {
            when { branch 'main' }
            steps {
                retry(3) {
                    timeout(time: 10, unit: 'MINUTES') {
                        nexusArtifactUploader(
                            nexusVersion:  'nexus3',
                            protocol:      'http',
                            nexusUrl:      'host.docker.internal:8081', // requires host-gateway mapping on Linux agents
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
        }

    }

    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
            slackSend(
                channel: '#ci-notifications',
                color:   'good',
                message: "BUILD PASSED: ${env.APP_NAME} v${env.APP_VERSION} | ${env.BUILD_URL}"
            )
            emailext(
                to:      'devteam@techbuild.io',
                subject: "BUILD PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:    "Successful build for ${env.APP_NAME} v${env.APP_VERSION}\nURL: ${env.BUILD_URL}"
            )
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
            slackSend(
                channel: '#ci-notifications',
                color:   'danger',
                message: "BUILD FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
            )
            emailext(
                to:      'devteam@techbuild.io',
                subject: "BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:    "Build ${env.BUILD_NUMBER} failed.\nConsole: ${env.BUILD_URL}console"
            )
        }
        unstable {
            slackSend(
                channel: '#ci-notifications',
                color:   'warning',
                message: "BUILD UNSTABLE: ${env.APP_NAME} #${env.BUILD_NUMBER} — test failures | ${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}
