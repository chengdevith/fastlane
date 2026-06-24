pipeline {
    agent any

    environment {
        ANDROID_HOME = '/opt/android-sdk'
        ANDROID_SDK_ROOT = '/opt/android-sdk'
        PATH = "${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Environment') {
            steps {
                sh 'pwd'
                sh 'ls -la'
                sh 'java -version'
                sh 'node -v'
                sh 'npm -v'
                sh 'ruby -v'
                sh 'bundle -v'
                sh 'adb --version'
                sh 'sdkmanager --version'
            }
        }
    }
}