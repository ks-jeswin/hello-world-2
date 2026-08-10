// Repository: https://github.com/ks-jeswin/hello-world-2.git
// Pipeline: Checkout → Build → Test → Quality Analysis → Archive → Notify
// ═══════════════════════════════════════════════════════════════════════════

pipeline {

    // ── Docker agent using pre-built Maven + JDK 17 ─────────────────────────
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17-alpine'
            args  '-v $HOME/.m2:/root/.m2'    // Cache Maven dependencies
        }
    }

    // ── Environment variables ───────────────────────────────────────────────
    environment {
        APP_NAME     = 'hello-world-2'
        APP_VERSION  = "1.0.${env.BUILD_NUMBER}"
        MAVEN_OPTS   = '-Xmx1024m -XX:+TieredCompilation'
        SONAR_URL    = 'http://sonarqube:9000'
        ARTIFACT_DIR = 'target'
    }

    // ── Pipeline-wide options ───────────────────────────────────────────────
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    // ── Triggers ────────────────────────────────────────────────────────────
    triggers {
        githubPush()
    }

    // ══════════════════════════════════════════════════════════════════════
    stages {

        // ── STAGE 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def commitHash = env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : 'UNKNOWN'
                    echo "Branch: ${env.GIT_BRANCH ?: 'default'} | Commit: ${commitHash}"
                }
            }
        }

        // ── STAGE 2: Build ────────────────────────────────────────────────
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

        // ── STAGE 3: Test ─────────────────────────────────────────────────
        stage('Test') {
            steps {
                sh 'mvn test -B'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: true
                }
                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'
                    script {
                        def results = currentBuild.rawBuild.getAction(hudson.tasks.test.AbstractTestResultAction.class)
                        if (results && results.totalCount > 0) {
                            def passRate = (results.totalCount - results.failCount) / results.totalCount * 100
                            if (passRate < 80) {
                                error("Test pass rate ${passRate.round(1)}% is below 80% threshold!")
                            }
                        }
                    }
                }
            }
        }

        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
        stage('Quality Analysis') {
            steps {
                script {
                    try {
                        withSonarQubeEnv('SonarQube-Local') {
                            sh """
                                mvn sonar:sonar \
                                  -Dsonar.projectKey=${env.APP_NAME} \
                                  -Dsonar.projectName="TechBuild ${env.APP_NAME}" \
                                  -Dsonar.projectVersion=${env.APP_VERSION} \
                                  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                                  -B
                            """
                        }
                    } catch (Exception e) {
                        echo "SonarQube step skipped/failed: ${e.getMessage()}"
                    }
                }
            }
        }

        // ── STAGE 5: Quality Gate ─────────────────────────────────────────
        stage('Quality Gate') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: false
                        }
                    } catch (Exception e) {
                        echo "Quality Gate check skipped/failed: ${e.getMessage()}"
                    }
                }
            }
        }

        // ── STAGE 6: Package & Archive ────────────────────────────────────
        stage('Package & Archive') {
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true, fingerprint: true
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.jar"
            }
        }

        // ── STAGE 7: Publish to Nexus ─────────────────────────────────────
        stage('Publish Artifact') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                script {
                    try {
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
                                file:       "target/${env.APP_NAME}-${env.APP_VERSION}.jar",
                                type:       'jar'
                            ]]
                        )
                    } catch (Exception e) {
                        echo "Nexus publication step skipped/failed: ${e.getMessage()}"
                    }
                }
            }
        }

    }    // end stages

    // ── Post-build actions ─────────────────────────────────────────────────
    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
            script {
                try {
                    slackSend(
                        channel: '#ci-notifications',
                        color:   'good',
                        message: "BUILD PASSED: ${env.APP_NAME} v${env.APP_VERSION} | ${env.BUILD_URL}"
                    )
                } catch (Exception e) {
                    echo "Slack notification skipped/failed: ${e.getMessage()}"
                }
            }
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
            script {
                try {
                    slackSend(
                        channel: '#ci-notifications',
                        color:   'danger',
                        message: "BUILD FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
                    )
                } catch (Exception e) {
                    echo "Slack notification skipped/failed: ${e.getMessage()}"
                }
            }
        }
        always {
            cleanWs()    // Clean workspace to free disk space
        }
    }

}    // end pipeline
