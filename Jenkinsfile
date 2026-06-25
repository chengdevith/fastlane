pipeline {
    agent { label 'mac' }

    environment {
        JAVA_HOME = '/Users/enz/Library/Java/JavaVirtualMachines/ms-21.0.10/Contents/Home'
        ANDROID_HOME = '/Users/enz/Library/Android/sdk'
        ANDROID_SDK_ROOT = '/Users/enz/Library/Android/sdk'
        PATH = "${JAVA_HOME}/bin:${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${env.PATH}"
        NODE_ENV = 'production'
    }

    stages {
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

        stage('Install Node Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Prepare Android') {
            steps {
                sh 'chmod +x android/gradlew'
                sh 'echo "sdk.dir=/Users/enz/Library/Android/sdk" > android/local.properties'
            }
        }

        stage('Install Fastlane Dependencies') {
            steps {
                dir('android') {
                    sh 'bundle config set path vendor/bundle'
                    sh 'bundle install'
                }
            }
        }

        stage('Build Android Release') {
            steps {
                dir('android') {
                    withCredentials([
                                    file(credentialsId: 'android-upload-keystore', variable: 'ANDROID_KEYSTORE_FILE'),
                                    string(credentialsId: 'android-keystore-password', variable: 'ANDROID_KEYSTORE_PASSWORD'),
                                    string(credentialsId: 'android-key-alias', variable: 'ANDROID_KEY_ALIAS'),
                                    string(credentialsId: 'android-key-password', variable: 'ANDROID_KEY_PASSWORD')
                                ]) {
                                    sh 'bundle exec fastlane build_release'
                                }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'android/app/build/outputs/**/*', allowEmptyArchive: true
        }

        success {
            echo 'Android build completed successfully.'
        }

        failure {
            echo 'Android build failed. Check console output.'
        }
    }
}