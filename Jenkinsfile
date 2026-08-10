// Repository: https://github.com/ks-jeswin/hello-world-2.git
// Pipeline: Checkout → Build → Test → Quality Analysis → Quality Gate → Package & Archive → Publish Artifact

pipeline {

    // ── Docker agent with Maven and Java pre-installed ───────────────────────
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17-alpine'
            args  '-v $HOME/.m2:/root/.m2'    // Cache Maven dependencies between builds
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

    // ── Build on push to any branch; deploy only from main ─────────────────
    triggers {
        githubPush()    // Requires GitHub plugin — responds to webhook events
    }

    // ══════════════════════════════════════════════════════════════════════
    stages {

        // ── STAGE 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH ?: 'main'} | Commit: ${env.GIT_COMMIT ? env.GIT_COMMIT[0..7] : 'HEAD'}"
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
                    // Publish JUnit test results regardless of pass/fail
                    junit(testResults: 'target/surefire-reports/**/*.xml',
                          allowEmptyResults: false)
                }
                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'
                    // Enforce 80% pass rate threshold:
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

        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
        stage('Quality Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                    withSonarQubeEnv('SonarQube-Local') {
                        sh """
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3585:sonar \
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

        // ── STAGE 5: Quality Gate ─────────────────────────────────────────
        stage('Quality Gate') {
            agent none
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── STAGE 6: Package & Archive ────────────────────────────────────
        stage('Package & Archive') {
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts(artifacts: 'target/*.jar', fingerprint: true)
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.jar"
            }
        }

        // ── STAGE 7: Publish to Nexus (main branch only) ─────────────────
        stage('Publish Artifact') {
            when { branch 'main' }
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
                        file:       "target/${env.APP_NAME}-${env.APP_VERSION}.jar",
                        type:       'jar'
                    ]]
                )
            }
        }

    }    // end stages

    // ── Post-build actions ─────────────────────────────────────────────────
    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
        }
        always {
            cleanWs()    // Clean workspace after every build to free disk space
        }
    }

}    // end pipeline
