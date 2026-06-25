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
            // Stop Gradle daemon safely
            dir('android') {
                sh './gradlew --stop || true'
            }

            // Copy AAB to Mac Desktop first
            sh '''
                mkdir -p "$HOME/Desktop/jenkins-artifacts/mobile-android"

                AAB_FILE=$(ls android/app/build/outputs/bundle/release/*.aab 2>/dev/null | head -1 || true)

                if [ -n "$AAB_FILE" ]; then
                    cp -f "$AAB_FILE" "$HOME/Desktop/jenkins-artifacts/mobile-android/app-release-${BUILD_NUMBER}.aab"
                    echo "AAB copied to Mac Desktop:"
                    ls -lh "$HOME/Desktop/jenkins-artifacts/mobile-android/"
                else
                    echo "No AAB file found."
                fi
            '''

            // Try to archive to Jenkins UI, but do not fail build if archive has remoting issue
            script {
                try {
                    archiveArtifacts artifacts: 'android/app/build/outputs/bundle/release/*.aab', allowEmptyArchive: false
                } catch (err) {
                    echo "archiveArtifacts failed, but Android release build already succeeded."
                    echo "AAB is available on the Mac agent Desktop."
                    echo "${err}"
                }
            }
        }

        success {
            echo 'Android release AAB build completed successfully.'
        }

        failure {
            echo 'Android build failed. Check console output.'
        }
    }
}