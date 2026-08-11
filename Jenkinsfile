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
                echo
